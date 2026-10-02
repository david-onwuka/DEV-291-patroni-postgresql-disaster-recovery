# DEV-291 — Restore Patroni PostgreSQL Cluster
## Overview
This project is a local PostgreSQL disaster-recovery and high-availability lab built with Docker Compose.
The project demonstrates:
- PostgreSQL 16- Patroni high availability- 3-node etcd distributed configuration store- pgBackRest physical backups- WAL archiving and recovery- Simulated PostgreSQL data loss- Full database restoration- Patroni leader recovery- Post-restore data validation- Linux and Docker troubleshooting
The core objective was to simulate a PostgreSQL disaster, restore the database from pgBackRest, recover the required WAL, and bring the restored database back under Patroni control.
---
## Architecture
                         +-----------------+                         |     HAProxy     |                         |    :5000/:5001  |                         +--------+--------+                                  |              +-------------------+-------------------+              |                   |                   |          +---+---+           +---+---+           +---+---+          |  pg1  |           |  pg2  |           |  pg3  |          |Patroni|           |Patroni|           |Patroni|          | PG16  |           | PG16  |           | PG16  |          +---+---+           +---+---+           +---+---+              |                   |                   |              +-------------------+-------------------+                                  |                         +--------+--------+                         |    etcd1/2/3    |                         |       DCS       |                         +-----------------+
                         +-----------------+                         |   pgBackRest    |                         | Backup + WAL    |                         |    Repository   |                         +-----------------+
---
## Disaster Recovery Workflow
Healthy Patroni cluster        |        vCreate full pgBackRest backup        |        vInsert restore-validation data        |        vCreate second full backup        |        vSimulate disaster        |        vStop PostgreSQL nodes        |        vRemove PostgreSQL data volumes        |        vRestore latest full backup        |        vRecover archived WAL        |        vPostgreSQL completes recovery        |        vPatroni re-establishes pg1 as Leader        |        vValidate recovered application data
---
## Project Directory
patroni-dev291/├── patroni/├── etcd/├── haproxy/├── pgbackrest/├── pgbackrest-repo/├── scripts/├── logs/├── docs/├── docker-compose.yml└── .env
---
## Core Components
### PostgreSQL
PostgreSQL 16 was used for all database nodes.
### Patroni
Patroni manages PostgreSQL high availability, leader election, replication topology, and failover.
Nodes:
- pg1- pg2- pg3
### etcd
A three-node etcd cluster was used as Patroni's distributed configuration store.
Nodes:
- etcd1- etcd2- etcd3
### pgBackRest
pgBackRest was used for:
- Full physical backups- WAL archiving- Backup validation- Physical database restore- Archived WAL retrieval
### HAProxy
HAProxy was included in the architecture for PostgreSQL traffic routing and health checking.
The core disaster-recovery acceptance test did not depend on HAProxy.
---
## Backup Configuration
The pgBackRest configuration used:
[global]repo1-path=/var/lib/pgbackrestrepo1-retention-full=2repo1-retention-diff=4repo1-type=posixlog-level-console=info
[patroni-dev291]pg1-path=/var/lib/postgresql/data
Repository:
/var/lib/pgbackrest
Successful full backups created during the test:
20261002-131029F20261002-131403F
The second backup contained the data used to validate the disaster-recovery process.
---
## Initial HA Validation
Before the disaster simulation, the Patroni cluster reached a healthy state:
pg1  Leaderpg2  Replicapg3  Replica
The replicas were streaming from the primary and the cluster was operating on timeline 4.
The etcd cluster was also healthy across all three members.
---
## Restore Validation Data
Before simulating the disaster, a dedicated validation table was created:
CREATE TABLE IF NOT EXISTS dev291_restore_test (    id SERIAL PRIMARY KEY,    message TEXT NOT NULL,    created_at TIMESTAMPTZ DEFAULT NOW());
INSERT INTO dev291_restore_test (message)VALUES ('DEV-291 restore validation data');
The validation record was created before the disaster.
The purpose was to prove that the restore recovered real database content rather than simply starting an empty PostgreSQL instance.
---
## Disaster Simulation
The PostgreSQL nodes were stopped.
The PostgreSQL Docker containers were removed.
The PostgreSQL data volumes were removed.
This simulated complete loss of the PostgreSQL data directories.
The pgBackRest repository was preserved.
The latest successful backup was:
20261002-131403F
---
## Restore Procedure
A clean pg1 PostgreSQL volume was created.
The 20261002-131403F backup was restored using pgBackRest.
Restore completed successfully:
restore backup set 20261002-131403Frestore size = 22.2MBfile total = 977restore command end: completed successfully
The restored database initially required archived WAL before recovery could complete.
---
## WAL Recovery
The pgBackRest repository contained archived WAL on timeline 4.
The required WAL segment was:
000000040000000000000007
The repository successfully returned the WAL segment using pgBackRest archive-get.
The WAL segment was then made available to PostgreSQL.
PostgreSQL continued recovery from the restored physical backup.
---
## Final Recovery
After WAL recovery, PostgreSQL reported:
database system is ready to accept read-only connections
Patroni acquired the session lock and promoted pg1:
promoted self to leader because I had the session lock
PostgreSQL completed archive recovery:
archive recovery complete
PostgreSQL selected a new timeline:
selected new timeline ID: 6
PostgreSQL then reported:
database system is ready to accept connections
The restored PostgreSQL instance was successfully returned to service under Patroni.
---
## Restore Validation
The restored data was queried with:
SELECT * FROM dev291_restore_test;
Result:
id |             message             |          created_at---+---------------------------------+-------------------------------1  | DEV-291 restore validation data | 2026-10-02 13:13:17.441581+00
The original row created before the simulated disaster was recovered successfully.
This is the primary restore acceptance test for the project.
---
## Actual Failures Encountered and Resolved
### Failure 1 — etcd image pull / DNS
The initial etcd image pull encountered a Docker/DNS resolution problem while accessing the container registry.
The image was subsequently pulled successfully.
The three-node etcd cluster then became healthy.
### Failure 2 — Patroni variable interpolation
The initial Patroni configuration attempted to use:
${PATRONI_NAME}
directly inside the Patroni configuration.
Patroni did not substitute the value as expected.
The resulting error included:
could not translate host name "${patroni_name}" to address
Resolution:
Separate explicit Patroni configuration files were created for:
pg1pg2pg3
### Failure 3 — PostgreSQL data-directory permissions
PostgreSQL initially rejected data directories because of invalid permissions.
The data directories were corrected to PostgreSQL's required restrictive permissions.
### Failure 4 — pgBackRest repository permissions
Initial stanza creation failed with:
unable to create path '/var/lib/pgbackrest/archive': [13] Permission denied
Resolution:
sudo chown -R 999:999 pgbackrest-repo
After correcting repository ownership, stanza creation succeeded.
### Failure 5 — WAL archiving disabled
The first full backup failed with:
archive_mode must be enabled
Resolution:
PostgreSQL was configured through Patroni with:
archive_mode=onarchive_command=pgbackrest --stanza=patroni-dev291 archive-push %p
The PostgreSQL nodes were restarted and subsequent full backups completed successfully.
### Failure 6 — Restore initially waiting for WAL
After the physical backup was restored, PostgreSQL remained in recovery waiting for WAL.
The requested WAL location was:
0/7000760
The pgBackRest repository was verified to contain the required WAL.
The required segment was:
000000040000000000000007
pgBackRest archive-get successfully retrieved the segment.
After the WAL was made available, PostgreSQL completed recovery and Patroni promoted pg1.
---
## Evidence Captured
### Screenshot 1 — Healthy HA Cluster
Filename:
DEV-291-01-HA-cluster-healthy.png
Shows the initial Patroni cluster with pg1 as Leader and pg2/pg3 as Replicas.
### Screenshot 2 — Initial pgBackRest Backup
Filename:
DEV-291-02-initial-backup.png
Shows successful pgBackRest backup information.
### Screenshot 3 — Restore Validation Backup
Filename:
DEV-291-03-validation-backup.png
Shows the second full backup containing the restore-validation dataset.
Additional restore/recovery evidence can be added during final portfolio packaging.
---
## Useful Commands
Check Patroni cluster state:
docker exec patroni-pg1 patronictl -c /etc/patroni/patroni.yml list
Check pgBackRest:
docker exec patroni-pg1 pgbackrest --stanza=patroni-dev291 info
Check PostgreSQL:
docker exec patroni-pg1 psql -U postgres -c "SELECT version();"
Validate restored data:
docker exec patroni-pg1 psql -U postgres -c "SELECT * FROM dev291_restore_test;"
Retrieve archived WAL:
docker exec patroni-pg1 pgbackrest --stanza=patroni-dev291 archive-get 000000040000000000000007 /tmp/000000040000000000000007
Check etcd cluster health:
docker exec patroni-etcd1 etcdctl --endpoints=http://etcd1:2379,http://etcd2:2379,http://etcd3:2379 endpoint health --cluster
---
## Skills Demonstrated
- PostgreSQL 16 administration- Patroni high availability- PostgreSQL physical backup and recovery- pgBackRest- WAL archiving- WAL recovery- etcd- Docker- Docker Compose- Linux- Database disaster recovery- High-availability troubleshooting- Root-cause analysis- Recovery validation- Infrastructure configuration
---
## Key Outcome
The project successfully demonstrated an end-to-end PostgreSQL disaster-recovery workflow:
1. A healthy Patroni PostgreSQL environment was created.2. A three-node etcd DCS was established.3. PostgreSQL replication was established across the Patroni cluster.4. pgBackRest full backups were created.5. Real validation data was inserted.6. A second full backup captured the validation data.7. PostgreSQL data volumes were deliberately destroyed.8. The latest full backup was restored.9. Required archived WAL was recovered.10. PostgreSQL completed archive recovery.11. Patroni re-established pg1 as the Leader.12. The original validation record was successfully recovered.
---
## Project Status
DEV-291 — Core disaster-recovery objective: COMPLETE.
The project demonstrates practical experience with:
- PostgreSQL HA- Patroni- etcd- pgBackRest- WAL recovery- Docker- Docker Compose- Linux troubleshooting- Disaster recovery- Backup and restore validation- Root-cause analysis
The optional post-restore replica rejoin/failover validation was not required for the core disaster-recovery acceptance objective.
