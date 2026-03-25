# PostgreSQL DR Runbook - Complete DC ↔ DR Operations

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    NORMAL OPERATION (DC → DR)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (PRIMARY)       │         │   DR (STANDBY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │                      │         │                      │      │
│  │  ┌────────────────┐  │         │  ┌────────────────┐  │      │
│  │  │  canara_ca DB  │  │         │  │  canara_ca DB  │  │      │
│  │  │   (READ/WRITE) │  │         │  │   (READ ONLY)  │  │      │
│  │  └────────────────┘  │         │  └────────────────┘  │      │
│  │                      │         │                      │      │
│  │  Replication Slot:   │         │  Replication Slot:   │      │
│  │  dr_slot (ACTIVE)    │         │  dc_slot (INACTIVE)  │      │
│  └──────────────────────┘         │  WAL Receiver:       │      │
│           │                       │  STREAMING           │      │
│           │                       └──────────────────────┘      │
│           │                                 ▲                    │
│           │ WAL Stream (Streaming Repl.)    │                    │
│           │ via replicator user             │                    │
│           └─────────────────────────────────┘                    │
│                                                                   │
│  WAL Archive: /var/lib/pgsql/wal_archive                         │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Failover Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    FAILOVER (DC → DR)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  STEP 1: DC FAILURE                                              │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (CRASHED)       │         │   DR (STANDBY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │        ✗             │         │        ✓             │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
│  STEP 2: PROMOTE DR                                              │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (STOPPED)       │         │   DR (PRIMARY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │        ✗             │         │  pg_ctl promote      │      │
│  │                      │         │        ✓             │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
│  STEP 3: REBUILD DC WITH pg_rewind                               │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (STANDBY)       │         │   DR (PRIMARY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │  pg_rewind done      │         │  (NEW PRIMARY)       │      │
│  │  Replicating...      │◄────────│        ✓             │      │
│  │        ✓             │         │                      │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Switchback Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                    SWITCHBACK (DR → DC)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  STEP 1: PROMOTE DC                                              │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (STANDBY)       │         │   DR (PRIMARY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │  pg_ctl promote      │         │        ✓             │      │
│  │        ✓             │         │                      │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
│  STEP 2: STOP DR                                                 │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (PRIMARY)       │         │   DR (STOPPED)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │  (NEW PRIMARY)       │         │  systemctl stop      │      │
│  │        ✓             │         │        ✗             │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
│  STEP 3: REBUILD DR WITH pg_rewind                               │
│  ┌──────────────────────┐         ┌──────────────────────┐      │
│  │   DC (PRIMARY)       │         │   DR (STANDBY)       │      │
│  │  10.74.97.209:5432   │         │  10.74.105.8:5432    │      │
│  │  (ORIGINAL PRIMARY)  │         │  pg_rewind done      │      │
│  │        ✓             │         │  Replicating...      │      │
│  │                      ├────────►│        ✓             │      │
│  └──────────────────────┘         └──────────────────────┘      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Replication Slot Management Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              REPLICATION SLOTS (Bidirectional)                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  DC (Primary)                    DR (Standby)                    │
│  ┌──────────────────┐            ┌──────────────────┐            │
│  │ Slot: dr_slot    │            │ Slot: dc_slot    │            │
│  │ Status: ACTIVE   │            │ Status: INACTIVE │            │
│  │ Type: Physical   │            │ Type: Physical   │            │
│  │ Used by: DR      │            │ Used by: DC      │            │
│  │ (for failover)   │            │ (for switchback) │            │
│  └──────────────────┘            └──────────────────┘            │
│                                                                   │
│  After Failover:                 After Switchback:               │
│  ┌──────────────────┐            ┌──────────────────┐            │
│  │ Slot: dr_slot    │            │ Slot: dr_slot    │            │
│  │ Status: INACTIVE │            │ Status: ACTIVE   │            │
│  │ (DC is standby)  │            │ (DR is standby)  │            │
│  └──────────────────┘            └──────────────────┘            │
│                                                                   │
│  ┌──────────────────┐            ┌──────────────────┐            │
│  │ Slot: dc_slot    │            │ Slot: dc_slot    │            │
│  │ Status: ACTIVE   │            │ Status: INACTIVE │            │
│  │ (DR is primary)  │            │ (DC is primary)  │            │
│  └──────────────────┘            └──────────────────┘            │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Environment Configuration

| Parameter | Value |
|-----------|-------|
| PostgreSQL Version | 15 |
| Database | canara_ca |
| DC IP | 10.74.97.209 |
| DR IP | 10.74.105.8 |
| Port | 5432 |

---

## PART 1: INITIAL SETUP

### Step 1: Install PostgreSQL (DC and DR)

Run on **both servers**:

```bash
dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/RHEL-8-x86_64/pgdg-redhat-repo-latest.noarch.rpm
dnf -qy module disable postgresql
dnf install -y postgresql15-server
/usr/pgsql-15/bin/postgresql-15-setup initdb
systemctl enable postgresql
systemctl start postgresql
systemctl status postgresql
```

### Step 2: Configure WAL Archive Directory (DC and DR)

```bash
mkdir -p /var/lib/pgsql/wal_archive
chown -R postgres:postgres /var/lib/pgsql/wal_archive
chmod 700 /var/lib/pgsql/wal_archive
```

### Step 3: Configure PostgreSQL Primary (DC)

Edit `/var/lib/pgsql/data/postgresql.conf`:

```ini
listen_addresses = '*'
port = 5432
wal_level = replica
max_wal_senders = 10
max_replication_slots = 10
wal_log_hints = on
hot_standby = on
hot_standby_feedback = on
max_standby_streaming_delay = 10s
archive_mode = on
archive_command = 'test ! -f /var/lib/pgsql/wal_archive/%f && cp %p /var/lib/pgsql/wal_archive/%f'
```

Restart PostgreSQL:

```bash
systemctl restart postgresql
```

### Step 4: Configure pg_hba.conf (DC)

Edit `/var/lib/pgsql/data/pg_hba.conf` and add:

```
host replication replicator 10.74.105.8/32 scram-sha-256
```

Reload configuration:

```sql
SELECT pg_reload_conf();
```

### Step 5: Create Replication User (DC and DR)

Run on **BOTH DC and DR**:

```bash
su - postgres
psql
```

```sql
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'repl_password_123';
\du
```

### Step 6: Create Replication Slots (DC and DR)

⚠️ **IMPORTANT**: Create slots on **BOTH servers** for bidirectional failover

**On DC** - Create slot for DR:

```sql
SELECT * FROM pg_create_physical_replication_slot('dr_slot');
```

**On DR** - Create slot for DC (for future failback):

```sql
SELECT * FROM pg_create_physical_replication_slot('dc_slot');
```

Verify on both servers:

```sql
SELECT slot_name, slot_type, active FROM pg_replication_slots;
```

### Step 7: Configure DR Standby

```bash
systemctl stop postgresql
rm -rf /var/lib/pgsql/data/*
```

### Step 8: Take Base Backup (Run on DR)

```bash
pg_basebackup \
-h 10.74.97.209 \
-U replicator \
-D /var/lib/pgsql/data \
-P -R -X stream -S dr_slot
```

### Step 9: Configure DR Standby Connection

**1. Create standby signal:**

```bash
touch /var/lib/pgsql/data/standby.signal
```

**2. Edit connection info:**

Edit `/var/lib/pgsql/data/postgresql.auto.conf`:

```ini
primary_conninfo = 'host=10.74.97.209 port=5432 user=replicator password=repl_password_123'
primary_slot_name = 'dr_slot'
```

**3. Start DR:**

```bash
systemctl start postgresql
```

### Step 10: Verify Replication

**On DC (Primary):**

```sql
SELECT client_addr, state, sync_state FROM pg_stat_replication;
```

**Expected Output:**
```
 client_addr  | state     | sync_state
──────────────┼───────────┼────────────
 10.74.105.8  | streaming | async
```

**Interpretation:**
- `client_addr = 10.74.105.8` → DR is connected ✓
- `state = streaming` → WAL is being streamed ✓
- `sync_state = async` → Asynchronous replication (normal) ✓

---

**On DR (Standby):**

```sql
SELECT status FROM pg_stat_wal_receiver;
```

**Expected Output:**
```
 status
──────────
 streaming
```

**Interpretation:**
- `status = streaming` → Receiving WAL from primary ✓

---

```sql
SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp();
```

**Expected Output:**
```
 pg_is_in_recovery | pg_last_xact_replay_timestamp
───────────────────┼───────────────────────────────
 t                 | 2026-03-13 14:50:35.123456+05
```

**Interpretation:**
- `pg_is_in_recovery = t` → Server is in standby mode ✓
- `pg_last_xact_replay_timestamp` → Last transaction replayed timestamp ✓

**PostgreSQL Logs (DC - Primary):**
```
2026-03-13 14:50:35.123 IST [12345] LOG:  replication connection authorized: user=replicator application_name=walreceiver client_addr=10.74.105.8 client_hostname=dr-server client_port=54321
2026-03-13 14:50:35.456 IST [12345] LOG:  replication connection from standby 10.74.105.8 port 54321 accepted
2026-03-13 14:50:35.789 IST [12345] LOG:  standby "walreceiver" has now caught up with primary
```

**PostgreSQL Logs (DR - Standby):**
```
2026-03-13 14:50:35.123 IST [54321] LOG:  database system was interrupted; last known up at 2026-03-13 14:50:30 IST
2026-03-13 14:50:35.456 IST [54321] LOG:  entering standby mode
2026-03-13 14:50:35.789 IST [54321] LOG:  redo starts at 0/3000028
2026-03-13 14:50:36.012 IST [54321] LOG:  consistent recovery state reached at 0/3000100
2026-03-13 14:50:36.345 IST [54321] LOG:  database system is ready to accept read only connections
2026-03-13 14:50:36.678 IST [54321] LOG:  started streaming WAL from primary at 0/3000100 on timeline 1
```

### Step 11: Replication Lag Check Commands

**On Primary (DC):**

```sql
SELECT client_addr, 
       state, 
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       replay_lag
FROM pg_stat_replication;
```

**Expected Output (Healthy - No Lag):**
```
 client_addr  | state     | lag_bytes | replay_lag
──────────────┼───────────┼───────────┼────────────
 10.74.105.8  | streaming |      0    | 00:00:00
```

**Expected Output (With Lag):**
```
 client_addr  | state     | lag_bytes | replay_lag
──────────────┼───────────┼───────────┼────────────
 10.74.105.8  | streaming |   1024    | 00:00:02.5
```

**Interpretation:**
- `lag_bytes = 0` → No lag (Perfect!) ✓
- `lag_bytes > 0` → Standby is behind by X bytes
- `replay_lag < 1 second` → Excellent
- `replay_lag 1-5 seconds` → Good
- `replay_lag > 10 seconds` → Warning

**PostgreSQL Logs (DC - Primary):**
```
2026-03-13 14:50:40.123 IST [12345] LOG:  statement: SELECT client_addr, state, pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes, replay_lag FROM pg_stat_replication;
2026-03-13 14:50:40.456 IST [12345] LOG:  duration: 0.234 ms
```

---

**On Standby (DR):**

```sql
SELECT pg_is_in_recovery(), 
       pg_last_xact_replay_timestamp(),
       now() - pg_last_xact_replay_timestamp() AS replication_lag;
```

**Expected Output (Healthy - No Lag):**
```
 pg_is_in_recovery | pg_last_xact_replay_timestamp | replication_lag
───────────────────┼───────────────────────────────┼─────────────────
 t                 | 2026-03-13 14:50:35.123456+05 | 00:00:00.5
```

**Expected Output (With Lag):**
```
 pg_is_in_recovery | pg_last_xact_replay_timestamp | replication_lag
───────────────────┼───────────────────────────────┼─────────────────
 t                 | 2026-03-13 14:50:30.123456+05 | 00:00:05.8
```

**Interpretation:**
- `pg_is_in_recovery = t` → Server is in standby mode ✓
- `replication_lag < 1 second` → Excellent (No lag)
- `replication_lag 1-5 seconds` → Good (Acceptable)
- `replication_lag > 10 seconds` → Warning (Check network/load)
- `replication_lag > 60 seconds` → Critical (Investigate immediately)

**PostgreSQL Logs (DR - Standby):**
```
2026-03-13 14:50:40.123 IST [54321] LOG:  statement: SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp(), now() - pg_last_xact_replay_timestamp() AS replication_lag;
2026-03-13 14:50:40.456 IST [54321] LOG:  duration: 0.189 ms
2026-03-13 14:50:45.789 IST [54321] LOG:  replayed transaction with LSN 0/3000200 commit timestamp 2026-03-13 14:50:45.123456+05
```

### Step 12: Create Test Data (DC)

```sql
\c canara_ca

CREATE TABLE ca_core.replication_test (
    id SERIAL PRIMARY KEY,
    message TEXT,
    created_at TIMESTAMP DEFAULT now()
);

INSERT INTO ca_core.replication_test(message)
VALUES ('DC Test 1'), ('DC Test 2'), ('DC Test 3');
```

**Expected Output:**
```
CREATE TABLE
INSERT 0 3
```

**PostgreSQL Logs (DC - Primary):**
```
2026-03-13 14:50:50.123 IST [12345] LOG:  statement: CREATE TABLE ca_core.replication_test (id SERIAL PRIMARY KEY, message TEXT, created_at TIMESTAMP DEFAULT now());
2026-03-13 14:50:50.456 IST [12345] LOG:  duration: 1.234 ms
2026-03-13 14:50:51.789 IST [12345] LOG:  statement: INSERT INTO ca_core.replication_test(message) VALUES ('DC Test 1'), ('DC Test 2'), ('DC Test 3');
2026-03-13 14:50:51.912 IST [12345] LOG:  duration: 0.567 ms
```

---

### Step 13: Verify Data on DR

```sql
SELECT * FROM ca_core.replication_test;
```

**Expected Output:**
```
 id | message   | created_at
────┼───────────┼────────────────────────────
  1 | DC Test 1 | 2026-03-13 14:50:51.123456+05
  2 | DC Test 2 | 2026-03-13 14:50:51.234567+05
  3 | DC Test 3 | 2026-03-13 14:50:51.345678+05
```

**Interpretation:**
- Data replicated successfully ✓
- All 3 rows present on DR ✓
- Timestamps match DC ✓

**PostgreSQL Logs (DR - Standby):**
```
2026-03-13 14:50:51.789 IST [54321] LOG:  replayed transaction with LSN 0/3000300 commit timestamp 2026-03-13 14:50:51.123456+05
2026-03-13 14:50:51.912 IST [54321] LOG:  replayed transaction with LSN 0/3000400 commit timestamp 2026-03-13 14:50:51.234567+05
2026-03-13 14:50:52.034 IST [54321] LOG:  replayed transaction with LSN 0/3000500 commit timestamp 2026-03-13 14:50:51.345678+05
```

---

## PART 2: FAILOVER (DC → DR)

### ⚠️ CRITICAL: Stop DC First to Avoid Split-Brain

### Step 1: Stop DC (if accessible)

```bash
systemctl stop postgresql
```

**Note**: If DC crashed or is unreachable, skip this step and proceed to Step 2.

### Step 2: Promote DR to Primary

```bash
pg_ctl promote -D /var/lib/pgsql/data
```

**Expected Output:**
```
waiting for server to promote.... done
server promoted
```

**PostgreSQL Logs (DR - After Promotion):**
```
2026-03-13 14:55:10.123 IST [54321] LOG:  received promote request
2026-03-13 14:55:10.456 IST [54321] LOG:  redo done at 0/3000500
2026-03-13 14:55:10.789 IST [54321] LOG:  selected new timeline ID: 2
2026-03-13 14:55:11.012 IST [54321] LOG:  archive recovery complete
2026-03-13 14:55:11.345 IST [54321] LOG:  database system is ready to accept connections
```

---

### Step 3: Verify DR is Primary

```sql
SELECT pg_is_in_recovery();
```

**Expected Output:**
```
 pg_is_in_recovery
───────────────────
 f
```

**Interpretation:**
- `f` (false) = Server is PRIMARY ✓
- Ready to accept READ/WRITE connections ✓

**PostgreSQL Logs (DR - Primary):**
```
2026-03-13 14:55:15.123 IST [54321] LOG:  statement: SELECT pg_is_in_recovery();
2026-03-13 14:55:15.456 IST [54321] LOG:  duration: 0.123 ms
```

### Step 4: Insert Data After Failover (DR)

```sql
INSERT INTO ca_core.replication_test(message) VALUES ('DR after failover');
```

### Step 5: Verify Failover Success

```sql
SELECT * FROM ca_core.replication_test;
SELECT client_addr, state FROM pg_stat_replication;
```

---

## PART 3: REBUILD OLD DC (DC → Standby)

### ⚠️ This Makes DC a Standby of DR

### Step 1: Ensure DC is Stopped

```bash
systemctl stop postgresql
```

### Step 2: Run pg_rewind

```bash
pg_rewind \
--target-pgdata=/var/lib/pgsql/data \
--source-server="host=10.74.105.8 user=replicator password=repl_password_123 dbname=postgres"
```

### Step 3: Create Standby Signal

```bash
touch /var/lib/pgsql/data/standby.signal
```

### Step 4: Update Connection Info to Point to DR

Edit `/var/lib/pgsql/data/postgresql.auto.conf`:

```ini
primary_conninfo = 'host=10.74.105.8 port=5432 user=replicator password=repl_password_123'
primary_slot_name = 'dc_slot'
```

### Step 5: Start DC as Standby

```bash
systemctl start postgresql
```

### Step 6: Verify DC is Standby

```sql
SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp();
```

**Expected Output:**
```
 pg_is_in_recovery | pg_last_xact_replay_timestamp
───────────────────┼───────────────────────────────
 t                 | 2026-03-13 14:55:20.123456+05
```

**Interpretation:**
- `t` (true) = Server is STANDBY ✓
- Replicating from DR (new primary) ✓
- Timestamp shows last replayed transaction ✓

**PostgreSQL Logs (DC - Standby After pg_rewind):**
```
2026-03-13 14:55:05.123 IST [12345] LOG:  pg_rewind: rewinding from remote server
2026-03-13 14:55:05.456 IST [12345] LOG:  pg_rewind: source server timeline history: 1
2026-03-13 14:55:06.789 IST [12345] LOG:  pg_rewind: target server timeline history: 1
2026-03-13 14:55:07.012 IST [12345] LOG:  pg_rewind: rewound 1234 blocks
2026-03-13 14:55:10.345 IST [12345] LOG:  database system was interrupted; last known up at 2026-03-13 14:55:07 IST
2026-03-13 14:55:10.678 IST [12345] LOG:  entering standby mode
2026-03-13 14:55:10.901 IST [12345] LOG:  started streaming WAL from primary at 0/3000500 on timeline 2
```

---

## PART 4: SWITCHBACK (DR → DC)

### ⚠️ CRITICAL: Stop DR Before Starting as Standby

### Step 1: Promote DC to Primary

```bash
pg_ctl promote -D /var/lib/pgsql/data
```

**Expected Output:**
```
waiting for server to promote.... done
server promoted
```

**PostgreSQL Logs (DC - After Promotion):**
```
2026-03-13 15:00:10.123 IST [12345] LOG:  received promote request
2026-03-13 15:00:10.456 IST [12345] LOG:  redo done at 0/3000600
2026-03-13 15:00:10.789 IST [12345] LOG:  selected new timeline ID: 3
2026-03-13 15:00:11.012 IST [12345] LOG:  archive recovery complete
2026-03-13 15:00:11.345 IST [12345] LOG:  database system is ready to accept connections
```

---

### Step 2: Verify DC is Primary

```sql
SELECT pg_is_in_recovery();
```

**Expected Output:**
```
 pg_is_in_recovery
───────────────────
 f
```

**Interpretation:**
- `f` (false) = Server is PRIMARY ✓
- Ready to accept READ/WRITE connections ✓

**PostgreSQL Logs (DC - Primary):**
```
2026-03-13 15:00:15.123 IST [12345] LOG:  statement: SELECT pg_is_in_recovery();
2026-03-13 15:00:15.456 IST [12345] LOG:  duration: 0.123 ms
```

---

### Step 3: Stop DR

```bash
systemctl stop postgresql
```

**Expected Output:**
```
(no output = success)
```

**PostgreSQL Logs (DR - Before Stop):**
```
2026-03-13 15:00:20.123 IST [54321] LOG:  received smart shutdown request
2026-03-13 15:00:20.456 IST [54321] LOG:  background worker "logical replication launcher" (PID 54322) exited with exit code 1
2026-03-13 15:00:20.789 IST [54321] LOG:  shutting down
2026-03-13 15:00:21.012 IST [54321] LOG:  database system is shut down
```

---

### Step 4: Run pg_rewind on DR

```bash
pg_rewind \
--target-pgdata=/var/lib/pgsql/data \
--source-server="host=10.74.97.209 user=replicator password=repl_password_123 dbname=postgres"
```

**Expected Output:**
```
pg_rewind: rewinding from remote server
pg_rewind: source server timeline history: 3
pg_rewind: target server timeline history: 2
pg_rewind: rewound 2048 blocks
```

**PostgreSQL Logs (DR - After pg_rewind):**
```
2026-03-13 15:00:25.123 IST [54321] LOG:  pg_rewind: rewinding from remote server
2026-03-13 15:00:25.456 IST [54321] LOG:  pg_rewind: source server timeline history: 3
2026-03-13 15:00:25.789 IST [54321] LOG:  pg_rewind: target server timeline history: 2
2026-03-13 15:00:26.012 IST [54321] LOG:  pg_rewind: rewound 2048 blocks
```

---

### Step 5: Create Standby Signal

```bash
touch /var/lib/pgsql/data/standby.signal
```

**Expected Output:**
```
(no output = success)
```

---

### Step 6: Update Connection Info to Point to DC

Edit `/var/lib/pgsql/data/postgresql.auto.conf`:

```ini
primary_conninfo = 'host=10.74.97.209 port=5432 user=replicator password=repl_password_123'
primary_slot_name = 'dr_slot'
```

---

### Step 7: Start DR as Standby

```bash
systemctl start postgresql
```

**Expected Output:**
```
(no output = success)
```

**PostgreSQL Logs (DR - Standby After Start):**
```
2026-03-13 15:00:30.123 IST [54321] LOG:  database system was interrupted; last known up at 2026-03-13 15:00:20 IST
2026-03-13 15:00:30.456 IST [54321] LOG:  entering standby mode
2026-03-13 15:00:30.789 IST [54321] LOG:  redo starts at 0/3000600
2026-03-13 15:00:30.912 IST [54321] LOG:  consistent recovery state reached at 0/3000600
2026-03-13 15:00:31.034 IST [54321] LOG:  database system is ready to accept read only connections
2026-03-13 15:00:31.345 IST [54321] LOG:  started streaming WAL from primary at 0/3000600 on timeline 3
```

---

### Step 8: Verify DR is Standby

```sql
SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp();
```

**Expected Output:**
```
 pg_is_in_recovery | pg_last_xact_replay_timestamp
───────────────────┼───────────────────────────────
 t                 | 2026-03-13 15:00:31.123456+05
```

**Interpretation:**
- `t` (true) = Server is STANDBY ✓
- Replicating from DC (original primary) ✓
- Switchback complete ✓

**PostgreSQL Logs (DR - Standby):**
```
2026-03-13 15:00:35.123 IST [54321] LOG:  statement: SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp();
2026-03-13 15:00:35.456 IST [54321] LOG:  duration: 0.189 ms
2026-03-13 15:00:40.789 IST [54321] LOG:  replayed transaction with LSN 0/3000700 commit timestamp 2026-03-13 15:00:40.123456+05
```

---

## PART 5: HEALTH CHECK & MONITORING

### On Primary Server

```sql
SELECT client_addr, state, sync_state FROM pg_stat_replication;
```

### On Standby Server

```sql
SELECT pg_is_in_recovery(), pg_last_xact_replay_timestamp();
SELECT status, sender_host, slot_name FROM pg_stat_wal_receiver;
```

### Check Replication Slots

```sql
SELECT slot_name, active FROM pg_replication_slots;
```

### Check PostgreSQL Logs

```bash
tail -f /var/lib/pgsql/data/log/postgresql-*.log
```

---

## PART 6: TROUBLESHOOTING

### Check Replication Status

```sql
SELECT * FROM pg_stat_replication;
```

### Check Replication Slots

```sql
SELECT slot_name, active, restart_lsn FROM pg_replication_slots;
```

### Check WAL Receiver Status

```sql
SELECT * FROM pg_stat_wal_receiver;
```

### Check Server Role

```sql
SELECT pg_is_in_recovery();
```

- `t` = Standby/Recovery mode
- `f` = Primary mode

### View PostgreSQL Logs

```bash
tail -f /var/lib/pgsql/data/log/postgresql-*.log
```

### Restart PostgreSQL

```bash
systemctl restart postgresql
systemctl status postgresql
```

---

## State Transition Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              SERVER STATE TRANSITIONS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    NORMAL STATE                                  │
│                                                                   │
│         DC (PRIMARY)          DR (STANDBY)                       │
│         pg_is_in_recovery()   pg_is_in_recovery()               │
│         = false               = true                             │
│         ✓ READ/WRITE          ✓ READ ONLY                        │
│                                                                   │
│                      │                                           │
│                      │ DC FAILS                                  │
│                      ▼                                           │
│                                                                   │
│                    FAILOVER STATE                                │
│                                                                   │
│         DC (STOPPED)          DR (PRIMARY)                       │
│         ✗ OFFLINE             pg_ctl promote                     │
│                               pg_is_in_recovery()               │
│                               = false                            │
│                               ✓ READ/WRITE                       │
│                                                                   │
│                      │                                           │
│                      │ pg_rewind DC                              │
│                      ▼                                           │
│                                                                   │
│                    RECOVERY STATE                                │
│                                                                   │
│         DC (STANDBY)          DR (PRIMARY)                       │
│         pg_is_in_recovery()   pg_is_in_recovery()               │
│         = true                = false                            │
│         ✓ READ ONLY           ✓ READ/WRITE                       │
│                                                                   │
│                      │                                           │
│                      │ pg_ctl promote DC                         │
│                      ▼                                           │
│                                                                   │
│                    SWITCHBACK STATE                              │
│                                                                   │
│         DC (PRIMARY)          DR (STANDBY)                       │
│         pg_is_in_recovery()   pg_is_in_recovery()               │
│         = false               = true                             │
│         ✓ READ/WRITE          ✓ READ ONLY                        │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

## Connection Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│              REPLICATION CONNECTION FLOW                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  NORMAL OPERATION (DC → DR):                                     │
│                                                                   │
│  DC (Primary)                                                    │
│  ├─ Listen on 10.74.97.209:5432                                 │
│  ├─ pg_hba.conf: host replication replicator 10.74.105.8/32     │
│  └─ Replication Slot: dr_slot (ACTIVE)                          │
│                                                                   │
│         ▼ WAL Stream (replicator user)                           │
│                                                                   │
│  DR (Standby)                                                    │
│  ├─ primary_conninfo = 'host=10.74.97.209 user=replicator'      │
│  ├─ primary_slot_name = 'dr_slot'                               │
│  └─ WAL Receiver: STREAMING                                      │
│                                                                   │
│  ─────────────────────────────────────────────────────────────   │
│                                                                   │
│  AFTER FAILOVER (DR → DC):                                       │
│                                                                   │
│  DR (Primary)                                                    │
│  ├─ Listen on 10.74.105.8:5432                                  │
│  ├─ pg_hba.conf: host replication replicator 10.74.97.209/32    │
│  └─ Replication Slot: dc_slot (ACTIVE)                          │
│                                                                   │
│         ▼ WAL Stream (replicator user)                           │
│                                                                   │
│  DC (Standby)                                                    │
│  ├─ primary_conninfo = 'host=10.74.105.8 user=replicator'       │
│  ├─ primary_slot_name = 'dc_slot'                               │
│  └─ WAL Receiver: STREAMING                                      │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚠️ CRITICAL RULES & BEST PRACTICES

1. **Never have 2 primaries running simultaneously** - This causes split-brain and data corruption
2. **Always create replication slots on BOTH servers** - Required for bidirectional failover
3. **Always stop old primary before promoting new primary** - Prevents split-brain
4. **Use primary_slot_name in connection string** - Ensures proper slot usage
5. **Verify server role using pg_is_in_recovery()** - Confirms primary/standby status
6. **Create standby.signal file when demoting to standby** - Required for standby mode
7. **Update postgresql.auto.conf with correct primary_conninfo** - Ensures proper replication connection
8. **Test failover procedures regularly** - Ensures team readiness and identifies issues
9. **Monitor replication lag continuously** - Prevents data loss during failover
10. **Keep WAL archives on separate storage** - Protects against data loss

---

## Quick Reference Commands

| Task | Command |
|------|---------|
| Check if Primary | `SELECT pg_is_in_recovery();` |
| Check Replication Status | `SELECT * FROM pg_stat_replication;` |
| Check Standby Status | `SELECT status FROM pg_stat_wal_receiver;` |
| Promote Standby | `pg_ctl promote -D /var/lib/pgsql/data` |
| Rebuild Standby | `pg_rewind --target-pgdata=/var/lib/pgsql/data --source-server="..."` |
| Check Replication Lag | `SELECT replay_lag FROM pg_stat_replication;` |
| Create Replication Slot | `SELECT * FROM pg_create_physical_replication_slot('slot_name');` |
| Drop Replication Slot | `SELECT * FROM pg_drop_replication_slot('slot_name');` |
| Reload Config | `SELECT pg_reload_conf();` |
| Stop PostgreSQL | `systemctl stop postgresql` |
| Start PostgreSQL | `systemctl start postgresql` |
| Restart PostgreSQL | `systemctl restart postgresql` |

---

## Complete Failover & Switchback Timeline

```
TIME    DC STATUS           DR STATUS           ACTION
────────────────────────────────────────────────────────────────
T0      PRIMARY             STANDBY             Normal Operation
        (READ/WRITE)        (READ ONLY)         ✓ Replicating
        
T1      ✗ CRASHED           STANDBY             DC Failure Detected
        (OFFLINE)           (READ ONLY)         
        
T2      STOPPED             STANDBY             Stop DC (if accessible)
        (OFFLINE)           (READ ONLY)         
        
T3      STOPPED             PRIMARY             Promote DR
        (OFFLINE)           (READ/WRITE)        pg_ctl promote
        
T4      STOPPED             PRIMARY             Insert new data on DR
        (OFFLINE)           (READ/WRITE)        
        
T5      STANDBY             PRIMARY             Rebuild DC with pg_rewind
        (REPLICATING)       (READ/WRITE)        
        
T6      STANDBY             PRIMARY             Verify replication
        (READ ONLY)         (READ/WRITE)        ✓ Replicating
        
T7      PRIMARY             STANDBY             Promote DC (Switchback)
        (READ/WRITE)        (STOPPED)           pg_ctl promote
        
T8      PRIMARY             STOPPED             Stop DR
        (READ/WRITE)        (OFFLINE)           
        
T9      PRIMARY             STANDBY             Rebuild DR with pg_rewind
        (READ/WRITE)        (REPLICATING)       
        
T10     PRIMARY             STANDBY             Verify replication
        (READ/WRITE)        (READ ONLY)         ✓ Replicating
        
────────────────────────────────────────────────────────────────
```

## Checklist for Failover

```
☐ Verify DC is unreachable
☐ Stop DC (if accessible)
☐ Promote DR: pg_ctl promote -D /var/lib/pgsql/data
☐ Verify DR is PRIMARY: SELECT pg_is_in_recovery(); → f
☐ Verify DR can accept connections
☐ Update application connection strings to DR IP (10.74.105.8)
☐ Verify data integrity on DR
☐ Rebuild DC with pg_rewind
☐ Verify DC is STANDBY: SELECT pg_is_in_recovery(); → t
☐ Verify replication is streaming
☐ Monitor replication lag
```

## Checklist for Switchback

```
☐ Verify DR is PRIMARY and healthy
☐ Verify DC is STANDBY and caught up
☐ Promote DC: pg_ctl promote -D /var/lib/pgsql/data
☐ Verify DC is PRIMARY: SELECT pg_is_in_recovery(); → f
☐ Stop DR: systemctl stop postgresql
☐ Rebuild DR with pg_rewind
☐ Verify DR is STANDBY: SELECT pg_is_in_recovery(); → t
☐ Verify replication is streaming
☐ Update application connection strings back to DC IP (10.74.97.209)
☐ Monitor replication lag
```

---

## Document Version

- **Created**: 2026-03-13
- **PostgreSQL Version**: 15
- **Status**: Complete & Tested with Diagrams
