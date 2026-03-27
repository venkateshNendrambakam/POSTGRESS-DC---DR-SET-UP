# PostgreSQL DC-DR Failover & Switchback SOP

## Environment
| Server | IP           | Port | Role (Normal) |
|--------|--------------|------|---------------|
| DC     | 10.32.51.81 | 4532 | PRIMARY       |
| DR     | 10.42.51.81  | 4532 | STANDBY       |

---

## SLOT RULE (Remember This)

> Slot is created on the PRIMARY. The Standby uses that slot name.

| Slot Name | Created On | Used By                  |
|-----------|------------|--------------------------|
| `dr_slot` | DC         | DR (when DR is standby)  |
| `dc_slot` | DR         | DC (when DC is standby)  |

---

## STEP 0 — Slot Check & Create (Always Do This Before Any Activity)

Run this step before both Failover and Switchback.

### On DC
```sql
psql -U canaradev -d canaradev -p 4532
SELECT slot_name, active FROM pg_replication_slots;
```

If `dr_slot` is missing → create it:
```sql
SELECT * FROM pg_create_physical_replication_slot('dr_slot');
```
### expected output
```
 slot_name | active
-----------+--------
 dr_slot   | f
(1 row)
```

### On DR
```sql
psql -U canaradev -d canaradev -p 4532
SELECT slot_name, active FROM pg_replication_slots;
```
If `dc_slot` is missing → create it:
```sql
SELECT * FROM pg_create_physical_replication_slot('dc_slot');
```

### expected output
```
 slot_name | active
-----------+--------
 dr_slot   | f
(1 row)
```

> If the slot already exists, skip creation — running it again will throw an error.

---

---

# SCENARIO 1 — FAILOVER (DC → DR)

**When:** DC has crashed or needs to be taken down. Promote DR as the new primary.

```
DC (PRIMARY)  ──►  DC (STANDBY)
DR (STANDBY)  ──►  DR (PRIMARY)
```

---

### PRE-CHECK

## Run in DC
```
select pg_is_in_recovery();
```
### In DC expected putput is:
```
 pg_is_in_recovery
-------------------
 f
(1 row)

```

## Run in DR
```
select pg_is_in_recovery();
```
### In DC expected putput is:
```
 pg_is_in_recovery
-------------------
 t
(1 row)

```


**On DC:**
```sql
SELECT pg_current_wal_lsn();
```

**On DR:**
```sql
SELECT pg_last_wal_receive_lsn();
```
Both values should be close — confirms DR is caught up.

---

### STEP 1 — Stop DC
**On DC:**
```bash
systemctl stop postgresql-15.service 
systemctl status postgresql-15.service
# Should show "inactive (dead)"
```
> If DC has crashed and is unreachable — skip this step.

---
>Note: Present scenario is DC is crashed or not reachable now DC is primary (No stand by signal in dc) ,  DR (standby having standby signal in data path). if DR is running as standby, standby signal definetly have it..You no need to check whether stand by signal is there or not in DR. Obviuosly it will be there of sure so now NO need to create a standby signal you can directly promote DR using below promote command(however DC is crashed).
---

### STEP 2 — Promote DR to Primary
**On DR:**
```bash
su - postgres
/usr/pgsql-15/bin/pg_ctl promote -D /var/lib/pgsql/15/data
```
Expected output:
```
waiting for server to promote.... done
server promoted
```

---

---
> Note 1: After promoting DR to primary the application can run immediatly on DR without pg_rewind or pg_basebackup, but DC "MUST NOT BE STARTED" untill its properlly synchronized to avoid split brain (If DC is started after dr is promoted without synchronization, both acts as primary, causes split brain an data inconsistency, which is dangerous)
---

---
> Note 2: Once you promoted DR then standby.signal file may or may not be deleted but it loses its significance because the server is no longer in standby mode

---

### STEP 3 — Verify DR is Primary
**On DR:**
```sql
psql -U canaradev -d canaradev -p 4532
SELECT pg_is_in_recovery();
-- Should return: f
```

---
> Note: For testing this poc lets add some records in any test table in DR, so that data difference will be there in DC and DR so that after hitting the rewind command you can find some sync percenatge.

---
**important point:**
### check all the files permission in data path /var/lib/pgsql/15/data (its should be of postgres) .If any one is of the file having other permission then entire data will get crashed after rewind command, which is VERY DANGEROUS.If in case Rewind command failed then we need to perform pg_basebackup activity. 
---

### STEP 4 — Run pg_rewind on DC
**Everything below is on DC:**

```bash
# Confirm DC is stopped
systemctl status postgresql-15.service

# Switch to postgres user
su - postgres

# Run pg_rewind
pg_rewind \
  --target-pgdata=/var/lib/pgsql/15/data \
  --source-server="host=10.42.51.81 port=4532 user=replicator password=very_secure_password dbname=postgres" \
  -P
```
Expected output:
```
pg_rewind: Done!
```

---

### STEP 5 — Configure DC as Standby
**On DC:**
```bash
# Create standby signal
touch /var/lib/pgsql/15/data/standby.signal

# Give postgres permissions to that standby.dignal file in the data path var/lib/pgsql/15/data/
chown postgres:postgres standby.dignal

# Update connection info
vi /var/lib/pgsql/15/data/postgresql.auto.conf
```
Content should be:
```ini
primary_conninfo = 'host=10.42.51.81 port=4532 user=replicator password=very_secure_password'
primary_slot_name = 'dc_slot'
```

---

### STEP 6 — Start DC
**On DC:**
```bash
systemctl start postgresql-15.service
```

---

### STEP 7 — Verify Replication
**On DC:**
```sql
SELECT pg_is_in_recovery();
-- Should return: t

SELECT status, sender_host, slot_name FROM pg_stat_wal_receiver;
-- status=streaming, sender_host=10.74.105.8, slot_name=dc_slot
```

**On DR:**
```sql
SELECT client_addr, state, replay_lag FROM pg_stat_replication;
-- client_addr=10.74.97.209, state=streaming, replay_lag=00:00:00
```

---

---

# SCENARIO 2 — SWITCHBACK (DR → DC)

**When:** After 1-2 months, bringing DC back as primary.

```
DC (STANDBY)  ──►  DC (PRIMARY)
DR (PRIMARY)  ──►  DR (STANDBY)
```

---

### PRE-CHECK

**On DR (current primary):**
```sql
SELECT client_addr, state, replay_lag FROM pg_stat_replication;
-- DC IP should be visible in streaming state, lag should be near zero
```

**On DC (current standby):**
```sql
SELECT pg_is_in_recovery();
-- Should return: t
```
> If lag is high — wait until it reaches zero before proceeding.

---

### STEP 1 — Promote DC to Primary
**On DC** (do NOT stop PostgreSQL before this — promote runs on a running server):
```bash
su - postgres
/usr/pgsql-15/bin/pg_ctl promote -D /var/lib/pgsql/15/data
```
Expected output:
```
waiting for server to promote.... done
server promoted
```

---

### STEP 2 — Verify DC is Primary
**On DC:**
```sql
psql -U canaradev -d canaradev -p 4532
SELECT pg_is_in_recovery();
-- Should return: f
```

---

### STEP 3 — Stop DR
**On DR:**
```bash
systemctl stop postgresql-15.service
systemctl status postgresql-15.service
# Should show "inactive (dead)"
```
> Stopping DR is mandatory before running pg_rewind — it cannot run on a live server.

---

### STEP 4 — Run pg_rewind on DR
**Everything below is on DR:**

---
>important point: check all the files permission in data path (its should be of postgres) .If any one is of the file having other permission then entire data will get crashed after rewind command
---

```bash
# Switch to postgres user
su - postgres

# Run pg_rewind

pg_rewind \
  --target-pgdata=/var/lib/pgsql/15/data \
  --source-server="host=10.32.51.81 port=4532 user=replicator password=very_secure_password dbname=postgres" \
  -P
```
Expected output:
```
pg_rewind: Done!
```

---

### STEP 5 — Configure DR as Standby
**On DR:**
```bash
# Create standby signal
touch /var/lib/pgsql/15/data/standby.signal
chown postgres:postgres standby.signal

# Update connection info
vi /var/lib/pgsql/15/data/postgresql.auto.conf
```
Content should be:
```ini
primary_conninfo = 'host=10.32.51.81 port=4532 user=replicator password=very_secure_password'
primary_slot_name = 'dr_slot'
```

---

### STEP 6 — Start DR
**On DR:**
systemctl start postgresql-15.service
```

---

### STEP 7 — Verify Replication
**On DR:**
```sql
SELECT pg_is_in_recovery();
-- Should return: t

SELECT status, sender_host, slot_name FROM pg_stat_wal_receiver;
-- status=streaming, sender_host=10.32.51.81, slot_name=dr_slot
```

**On DC:**
```sql
SELECT client_addr, state, replay_lag FROM pg_stat_replication;
-- client_addr=10.42.51.81, state=streaming, replay_lag=00:00:00
```

---

---

## Quick Reference

| Scenario        | DC Action              | DR Action              | pg_rewind Runs On |
|-----------------|------------------------|------------------------|-------------------|
| Failover        | stop → pg_rewind → standby | promote → primary  | DC                |
| Switchback      | promote → primary      | stop → pg_rewind → standby | DR            |

| State           | DC pg_is_in_recovery() | DR pg_is_in_recovery() | Active Slot       |
|-----------------|------------------------|------------------------|-------------------|
| Normal          | f (primary)            | t (standby)            | DC: dr_slot = t   |
| After Failover  | t (standby)            | f (primary)            | DR: dc_slot = t   |
| After Switchback| f (primary)            | t (standby)            | DC: dr_slot = t   |

---

## If pg_rewind Fails

```sql
-- Check on the primary server
SHOW wal_log_hints;   -- Should be: on
```

If `off`, use `pg_basebackup` instead:
```bash
# Run on the server being rebuilt as standby (clears data directory first)
rm -rf /var/lib/pgsql/15/data/*
pg_basebackup -h <PRIMARY_IP> -p 4532 -U replicator \
  -D /var/lib/pgsql/15/data -P -R -X stream -S <slot_name>
```
- Failover:   `<PRIMARY_IP>=10.42.51.81`,  `<slot_name>=dc_slot`
- Switchback: `<PRIMARY_IP>=10.32.51.81`, `<slot_name>=dr_slot`
