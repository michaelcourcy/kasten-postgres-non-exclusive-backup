# Goal 

Create a non exclusive postgres backup with Kasten.

# Backup workflow 

The backup workflow is inspired by this article ["Exclusive backup" method is deprecated - what now?](https://www.cybertec-postgresql.com/en/exclusive-backup-deprecated-what-now/) by Laurenz Albe.
And also from the [example blueprint](https://raw.githubusercontent.com/kanisterio/kanister/refs/heads/master/examples/postgresql/postgres-start-stop-blueprint.yaml) by 
Daniil Fedotov.

1. In the statefulset backupPrehook start a non exclusive posgres backup session with `SELECT pg_backup_start(label => 'kanister_backup', fast => false);` 
2. Let Kasten protect the PVC which is now in a consistent state for the backup 
3. In the statefulset backupPosthook stop the backup session with `SELECT * FROM pg_backup_stop(wait_for_archive => true);`


# Create a 16.0 postgres database 
## Deploy the bitnami helm chart 

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

kubectl create ns postgres-non-exclusive-backup
helm install postgres16 --namespace postgres-non-exclusive-backup bitnami/postgresql --version 13.2.4 
```

## Create some data 

```
kubectl exec -ti postgres16-postgresql-0 -n postgres-non-exclusive-backup -- bash
PGPASSWORD=${POSTGRES_PASSWORD} psql -U postgres
CREATE DATABASE test;
\c test
CREATE TABLE COMPANY(
     ID INT PRIMARY KEY     NOT NULL,
     NAME           TEXT    NOT NULL,
     AGE            INT     NOT NULL,
     ADDRESS        CHAR(50),
     SALARY         REAL,
     CREATED_AT    TIMESTAMP
);
INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,CREATED_AT) VALUES (10, 'Paul', 32, 'California', 20000.00, now());
INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,CREATED_AT) VALUES (20, 'Omkar', 32, 'California', 20000.00, now());
INSERT INTO COMPANY (ID,NAME,AGE,ADDRESS,SALARY,CREATED_AT) VALUES (30, 'Prasad', 32, 'California', 20000.00, now());
select * from company;
\q
exit
```

# Deploy the blueprint and test it 

## deploy the blueprint

Create it and bind it to the statefulset

```
kubectl create -f postgres-non-exclusive-backup.yaml 
kubectl annotate statefulset postgres16-postgresql kanister.kasten.io/blueprint='postgres-non-exclusive-backup' \
     --namespace=postgres-non-exclusive-backup
```

## Test 

1. Create a backup of the namespace 
2. delete the statefulset and the pvc 
3. Restore 
4. check you get your data back

In order to verify that all non exclusive backup operations were done as expected you can check the logs in the backup session pod 
before it is removed.

```
 kubectl logs postgres16-postgresql-backup-session
 pg_backup_start 
-----------------
 0/2000028
(1 row)

    lsn    |                           labelfile                           | spcmapfile 
-----------+---------------------------------------------------------------+------------
 0/2000138 | START WAL LOCATION: 0/2000028 (file 000000010000000000000002)+| 
           | CHECKPOINT LOCATION: 0/2000060                               +| 
           | BACKUP METHOD: streamed                                      +| 
           | BACKUP FROM: primary                                         +| 
           | START TIME: 2025-05-13 15:14:12 GMT                          +| 
           | LABEL: kanister_backup                                       +| 
           | START TIMELINE: 1                                            +| 
           |                                                               | 
```