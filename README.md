
# DEV-291: Restore a Patroni PostgreSQL Cluster

A local PostgreSQL disaster-recovery and high-availability lab built with Docker Compose.

*Stack:* PostgreSQL 16 · Patroni · 3-node etcd · pgBackRest · HAProxy

*Objective:* simulate a PostgreSQL disaster, restore from pgBackRest, recover the required WAL, and bring the database back under Patroni control.

## What This Project Demonstrates

- Patroni high availability with a 3-node etcd DCS
- pgBackRest physical backups and WAL archiving
- Simulated total data loss and full restoration
- Patroni leader recovery after restore
- Post-restore data validation
- Linux and Docker troubleshooting

---

## Architecture

text
                    +------------------+
                    |     HAProxy      |
                    |   :5000 / :5001  |
                    +--------+---------+
                             |
          +------------------+------------------+
          |                  |                  |
      +---+---+          +---+---+          +---+---+
      |  pg1  |          |  pg2  |          |  pg3  |
      |Patroni|          |Patroni|          |Patroni|
      | PG 16 |          | PG 16 |          | PG 16 |
      +---+---+          +---+---+          +---+---+
          |                  |                  |
          +------------------+------------------+
                             |
                    +--------+---------+
                    |   etcd1/2/3     |
                    |       DCS        |
                    +------------------+

                    +------------------+
                    |    pgBackRest    |
                    | Backup + WAL Repo|
                    +------------------+


## Disaster Recovery Workflow

text
Healthy Patroni cluster
        |
        v
Create full pgBackRest backup
        |
        v
Insert restore-validation data
        |
        v
Create second full backup
        |
        v
Simulate disaster (stop nodes, remove containers and data volumes)
        |
        v
Restore latest full backup
        |
        v
Recover archived WAL
        |
        v
PostgreSQL completes archive recovery
        |
        v
Patroni re-establishes pg1 as Leader
        |
        v
Validate recovered application data


## Project Layout

text
patroni-dev291/
├── patroni/
├── etcd/
├── haproxy/
├── pgbackrest/
├── pgbackrest-repo/
├── scripts/
├── logs/
├── docs/
├── docker-compose.yml
└── .env


---

## Components

| Component | Role |
|-----------|------|
| PostgreSQL 16 | Database engine on pg1, pg2, pg3 |
| Patroni | Leader election, replication topology, failover |
| etcd (etcd1/2/3) | Distributed configuration store for Patroni |
| pgBackRest | Full backups, WAL archiving, restore, WAL retrieval |
| HAProxy | Traffic routing and health checks (not required for the DR acceptance test) |

## Backup Configuration

ini
[global]
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=4
repo1-type=posix
log-level-console=info

[patroni-dev291]
pg1-path=/var/lib/postgresql/data


Full backups created during the test:

- 20261002-131029F
- 20261002-131403F (contains the validation data)

---

## Walkthrough

### 1. Initial HA validation

Before the disaster, the cluster was healthy on timeline 4:

| Member | Role |
|--------|------|
| pg1 | Leader |
| pg2 | Replica |
| pg3 | Replica |

Replicas were streaming from the primary and all three etcd members were healthy.

### 2. Restore validation data

sql
CREATE TABLE IF NOT EXISTS dev291_restore_test (
    id SERIAL PRIMARY KEY,
    message TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

INSERT INTO dev291_restore_test (message)
VALUES ('DEV-291 restore validation data');


This proves the restore recovered real content, not just an empty PostgreSQL instance.

### 3. Disaster simulation

- PostgreSQL nodes stopped
- Containers removed
- Data volumes removed
- pgBackRest repository *preserved*

### 4. Restore

A clean pg1 volume was created and backup 20261002-131403F was restored:

text
restore backup set 20261002-131403F
restore size = 22.2MB
file total = 977
restore command end: completed successfully


The restored database then needed archived WAL to finish recovery.

### 5. WAL recovery

The repository held archived WAL on timeline 4. The required segment was 000000040000000000000007, which pgbackrest archive-get retrieved successfully and made available to PostgreSQL.

### 6. Final recovery

text
database system is ready to accept read-only connections
promoted self to leader because I had the session lock
archive recovery complete
selected new timeline ID: 6
database system is ready to accept connections


pg1 returned to service as Leader under Patroni.

### 7. Validation

bash
docker exec patroni-pg1 psql -U postgres -c "SELECT * FROM dev291_restore_test;"


text
 id |             message             |          created_at
----+---------------------------------+-------------------------------
  1 | DEV-291 restore validation data | 2026-10-02 13:13:17.441581+00


The row created before the disaster was recovered. This is the primary acceptance test.

---

## Failures Encountered and Resolved

### 1. etcd image pull / DNS

Docker hit a DNS resolution problem pulling from the registry. The pull later succeeded and the 3-node etcd cluster became healthy.

### 2. Patroni variable interpolation

Using ${PATRONI_NAME} directly in the Patroni config was not substituted:

text
could not translate host name "${patroni_name}" to address


*Fix:* separate explicit config files for pg1, pg2 and pg3.

### 3. Data-directory permissions

PostgreSQL rejected data directories with invalid permissions. *Fix:* set the required restrictive permissions (0700).

### 4. pgBackRest repository permissions

text
unable to create path '/var/lib/pgbackrest/archive': [13] Permission denied


*Fix:*

bash
sudo chown -R 999:999 pgbackrest-repo


### 5. WAL archiving disabled

text
archive_mode must be enabled


*Fix:* configured through Patroni, then restarted the nodes:

text
archive_mode=on
archive_command=pgbackrest --stanza=patroni-dev291 archive-push %p


### 6. Restore stuck waiting for WAL

After restore, PostgreSQL stayed in recovery waiting for WAL at 0/7000760. The repository was verified to contain 000000040000000000000007, archive-get retrieved it, and PostgreSQL then completed recovery and Patroni promoted pg1.

---

## Useful Commands

Patroni cluster state:

bash
docker exec patroni-pg1 patronictl -c /etc/patroni/patroni.yml list


pgBackRest info:

bash
docker exec patroni-pg1 pgbackrest --stanza=patroni-dev291 info


PostgreSQL version:

bash
docker exec patroni-pg1 psql -U postgres -c "SELECT version();"


Validate restored data:

bash
docker exec patroni-pg1 psql -U postgres -c "SELECT * FROM dev291_restore_test;"


Retrieve archived WAL:

bash
docker exec patroni-pg1 pgbackrest --stanza=patroni-dev291 archive-get \
  000000040000000000000007 /tmp/000000040000000000000007


etcd health:

bash
docker exec patroni-etcd1 etcdctl \
  --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 \
  endpoint health --cluster


---

## Evidence

| # | File | Shows |
|---|------|-------|
| 1 | DEV-291-01-HA-cluster-healthy.png | Initial cluster: pg1 Leader, pg2/pg3 Replicas |
| 2 | DEV-291-02-initial-backup.png | Successful pgBackRest backup info |
| 3 | DEV-291-03-validation-backup.png | Second full backup containing the validation dataset |

## Outcome

1. Built a healthy Patroni PostgreSQL environment with a 3-node etcd DCS
2. Established replication across the cluster
3. Created pgBackRest full backups, including one with validation data
4. Destroyed the PostgreSQL data volumes
5. Restored the latest full backup and recovered the required WAL
6. PostgreSQL completed archive recovery and Patroni re-established pg1 as Leader
7. Recovered the original validation record

*Status:* DEV-291 core disaster-recovery objective: *COMPLETE*.

Optional post-restore replica rejoin/failover validation was not required for the core acceptance objective.

## Skills Demonstrated

PostgreSQL 16 administration · Patroni HA · etcd · pgBackRest · WAL archiving and recovery · Docker / Compose · Linux · disaster recovery · root-cause analysis · recovery validation
```
