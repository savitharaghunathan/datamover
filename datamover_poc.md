# Data Mover PoC

## Pre req
* Install VolSync
    * https://volsync.readthedocs.io/en/stable/installation/index.html 
* Install OADP operator (optional if you have velero installed already)
* Create a DPA CR  (optional if you have velero installed already)

## PoC 

### Backup

#### Step 1:
* Create a namespace for the PoC
    * ```kubectl create ns volsync-poc```
* Create the VolSync restic secret in that namespace
```
apiVersion: v1
kind: Secret
metadata:
  name: restic-config
type: Opaque
stringData:
  # The repository url
  RESTIC_REPOSITORY: s3:s3.amazonaws.com/<bucket>
  # The repository encryption key
  RESTIC_PASSWORD: <password>
  # ENV vars specific to the chosen back end
  # https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html
  AWS_ACCESS_KEY_ID: <access_id>
  AWS_SECRET_ACCESS_KEY: <access_key>
```

* More details for installing restic secret in [here]([https://volsync.readthedocs.io/en/stable/usage/restic/index.html#specifying-a-repository)

#### Step 2:
* Install an app that uses PVC. 
    * [example](https://volsync.readthedocs.io/en/stable/usage/restic/database_example.html#creating-source-pvc-to-be-backed-up)

* Add a new database 

```
$ kubectl exec --stdin --tty -n volsync-poc `kubectl get pods -n volsync-poc | grep mysql | awk '{print $1}'` -- /bin/bash

$ mysql -u root -p$MYSQL_ROOT_PASSWORD

> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.00 sec)


> create database volsync_poc;

> exit

$ exit
```


#### Step 3:
* Create a `ReplicationSource` definition 
```
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: database-source
  namespace: volsync-poc
spec:
  sourcePVC: mysql-pv-claim
  trigger:
    schedule: "*/30 * * * *"
  restic:
    pruneIntervalDays: 15
    repository: restic-config
    retain:
      hourly: 1
      daily: 1
      weekly: 1
      monthly: 1
      yearly: 1
    copyMethod: Snapshot
```

* ***Note: `gp2-csi` driver doesn’t support volume cloning. Make sure to use `Snapshot` as `copyMethod`.***


#### Step 4:
* Configure a velero backup in `openshift-adp` ns with snapshots set to false
```
apiVersion: velero.io/v1
kind: Backup
metadata:
  namespace: openshift-adp
  name: <backup_name>
spec:
  storageLocation: <bsl_name>
  includedNamespaces:
    - volsync-poc
  snapshotVolumes: false
```
* Wait for the backup to complete 

### Simulate a disaster

* Delete namespace `volsync-poc`

### Restore

#### Step 1:

* Create a namespace called `volsync-poc` 
    * ```kubectl create ns volsync-poc```
* Create the VolSync restic secret in that namespace
```
apiVersion: v1
kind: Secret
metadata:
  name: restic-config
type: Opaque
stringData:
  # The repository url
  RESTIC_REPOSITORY: s3:s3.amazonaws.com/<bucket>
  # The repository encryption key
  RESTIC_PASSWORD: <password>
  # ENV vars specific to the chosen back end
  # https://restic.readthedocs.io/en/stable/030_preparing_a_new_repo.html
  AWS_ACCESS_KEY_ID: <access_id>
  AWS_SECRET_ACCESS_KEY: <access_key>
```

* Create an empty PVC
```
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```

#### Step 2:

* Create a `ReplicationDestination` def in the `volsync-poc` namespace.

```
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: database-destination
spec:
  trigger:
    manual: restore
  restic:
    destinationPVC: mysql-pv-claim
    repository: restic-config
    copyMethod: None
```

* Create a velero restore from the backup that was taken before

```
apiVersion: velero.io/v1
kind: Restore
metadata:
  namespace: openshift-adp
  name: restore
spec:
  backupName: <backup_name>
```

* Verify the pods are up and running in `volsync-poc` ns
* Verify the list of databases has the entry we added before
```

$ kubectl exec --stdin --tty -n volsync-poc `kubectl get pods -n volsync-poc | grep mysql | awk '{print $1}'` -- /bin/bash
$ mysql -u root -p$MYSQL_ROOT_PASSWORD
> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |             
| sys                |
/ volsync-poc
+--------------------+
5 rows in set (0.00 sec)

> exit
$ exit
```