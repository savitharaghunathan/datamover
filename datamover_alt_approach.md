# Data Mover - Alternate design exploration


## Problem statement:
Avoid copying over secrets to namespaces other than openshift-adp. Initial design proposal suggested that we create restic secrets in user namespaces. THis might lead to security risk followed by quotas & restrictions.


## Solution 1

Use Velero's CSI plugin to take the snapshots in ns `foo` and move it over to `openshift-adp` namespace using [volume snapshot](https://kubernetes.io/docs/concepts/storage/volume-snapshots/) APIs. Run VolSync data mover process after that from `openshift-adp` namespace.


### Implementation details:
VolSync will be used as the Data Mover in this design. For now, VolSync is the only plugin that will be supported and `restic` will be the supported method for backup & restore of PVCs. Restic repository details are configured in a `secret` object which gets used by the VolSync's resources. This design takes advantage of VolSync's two resources - `ReplicationSource` & `ReplicationDestination`. `ReplicationSource` object helps with taking a backup of the PVCs and using restic to move it to the storage specified in the restic secret. `ReplicationDestination` object takes care of restoring the backup from the restic repository. There will be a 1:1 relationship between the replication src/dest CRs and PVCs.
```
...
DataMoverEnable: True/False
...
```

If the DataMover flag is enabled, then the user creates a restic secret with all the following details,
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
*Note: More details for installing restic secret in [here](https://volsync.readthedocs.io/en/stable/usage/restic/index.html#specifying-a-repository)*


Like any other plugins, DataMover plugin gets deployed as an init container in the velero deployment.

#### Backup
When a Velero backup is created, the data mover plugin creates a VolumeSnapShotBackup CR. 

VolumeSnapShot controller watches the cluster for the creation of VolumeSnapShotBackup CR. It manages the transfer of VolumeSnapshots from the user namespace to the protected namespace. Once the volumesnapshot has been transferred, the controller creates a PVC from that `volumesnapshot` resource and a dummy pod which mounts the newly created PVC object. The controller then creates a `ReplicationSource` CR. VolSync watches for the creation of `ReplicationSource` CR and copies the PVC data to the restic repository mentioned in the `restic-secret`.

A status controller is created to watch VolSync CRs. It watches the `ReplicationSource` and`ReplicationDestination` objects and updates VolumeSnapShot CR events. 

##### Sample CRs

##### VolumeSnapShotBackup CR

```
apiVersion: volumesnapshot.oadp.openshift.com/v1alpha1
kind: VolumeSnapshotBackup
metadata:
  name: <name>
  namespace: <app_namespace>
spec:
  volumeSnapshotContent:
    name: <snapshot_content_name>

```

##### ReplicationSource CR
```
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: database-source
  namespace: openshift-adp
spec:
  sourcePVC: <pvc_name>
  trigger:
    manual: <trigger_name>
  restic:
    pruneIntervalDays: 15
    repository: restic-config
    retain:
      hourly: 1
      daily: 1
      weekly: 1
      monthly: 1
      yearly: 1
    copyMethod: None
```
*Note: CopyMethod is set to `None`, since we have already transferred the snapshots and created a PVC using that snapshot.* 

#### Restore

When a velero restore object gets triggered, the controller looks for a `VolumeSnapShotBackup` resource in the backup resources. If it encounters one, then it creates a `VolumeSnapShotRestore` CR. VolumeSnapShot controller watches the cluster for the creation of VolumeSnapShotRestore CR. It then creates a `ReplicationDestination` CR. VolSync controller moves the data from the restic repo to a volumesnapshot in the protected namespace. Then the VolumeSnapShot controller will transfer the snapshot to the user namespace. Velero finishes restore process.

##### Sample CRs

##### VolumeSnapShotRestore CR

```
apiVersion: volumesnapshot.oadp.openshift.com/v1alpha1
kind: VolumeSnapshotRestore
metadata:
  name: <name>
  namespace: <app_namespace>
spec:
  destinationClaimRef:
    name: <PVC_claim_name>
    namespace: <namespace>
  resticSecret:
    name: <secret_name>
```

##### ReplicationDestination CR

```
apiVersion: volsync.backube/v1alpha1
kind: ReplicationDestination
metadata:
  name: <protected_namespace>
spec:
  trigger:
    manual: <trigger_name>
  restic:
    destinationPVC: <pvc_name>
    repository: restic-config
    copyMethod: None
```
#### Limitations
CSI controller is unaware of the duplicate volumesnapshotcontent pointer

---


## Solution 2
Use Velero's CSI plugin to take the snapshots in ns `foo` and move it over to `openshift-adp` namespace using Persistent Volume APIs. Run VolSync data mover process after that from `openshift-adp` namespace.


---


## open questions
* Restic secret has the info regarding restric repo. Do we have to create a new restic secret with a new repo for every VolSync CR?  
* Do we have to block velero during restore process? 
* How do we support partner data movers?
* Should our design support multiple data movers? 





---
### Solution 1 - Quick steps

1. Create a sample stateful app in ns `foo`. Take a velero backup using csi plugin. This creates a snapshot in namespace `foo`
    * https://github.com/openshift/oadp-operator/blob/master/docs/examples/csi_example.md
    * Take a look at the `volumesnapshot` & `volumesnapshotcontent` resource associated with the volumesnapshot.
```
 $ oc get volumesnapshot -n mssql-persistent
NAME                     READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT   RESTORESIZE   SNAPSHOTCLASS       SNAPSHOTCONTENT                                    CREATIONTIME   AGE
velero-mssql-pvc-fh5f5   true         mssql-pvc                           10Gi          example-snapclass   snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311   4d22h          4d22h
```
```
$ oc get volumesnapshotcontent | grep velero-mssql-pvc-fh5f5
snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311   true         10737418240   Retain           ebs.csi.aws.com   example-snapclass     velero-mssql-pvc-fh5f5                            4d22h
```
```
$ oc get volumesnapshotcontent snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311 -o yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  creationTimestamp: "2022-02-23T22:10:47Z"
  finalizers:
  - snapshot.storage.kubernetes.io/volumesnapshotcontent-bound-protection
  generation: 1
  labels:
    velero.io/backup-name: mssql-persistent
  name: snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311
  resourceVersion: "19821935"
  uid: 9b776de5-f0c6-4ae4-8d0f-ceaf4394b874
spec:
  deletionPolicy: Retain
  driver: ebs.csi.aws.com
  source:
    volumeHandle: vol-01f9fc6449d383392
  volumeSnapshotClassName: example-snapclass
  volumeSnapshotRef:
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    name: velero-mssql-pvc-fh5f5
    namespace: mssql-persistent
    resourceVersion: "19821379"
    uid: 5b5dda14-8a71-4c2b-acf7-56249aa46311
status:
  creationTime: 1645654250951000000
  readyToUse: true
  restoreSize: 10737418240
  snapshotHandle: snap-0f154973f29b185eb
```
2. Create a pre provisoned volumesnapshotcontent using the snapshot created in step 1. Make sure to refer the same `snapshotHandle`
```
$ cat staticvolsnapshotcontent.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: static-snapshot-content
spec:
  deletionPolicy: Retain
  driver: ebs.csi.aws.com
  source:
    snapshotHandle: snap-0f154973f29b185eb
  volumeSnapshotRef:
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    name: static-snapshot-demo
    namespace: openshift-adp
```
```
$ oc get VolumeSnapshotContent static-snapshot-content
NAME                      READYTOUSE   RESTORESIZE   DELETIONPOLICY   DRIVER            VOLUMESNAPSHOTCLASS   VOLUMESNAPSHOT         AGE
static-snapshot-content   true         10737418240   Retain           ebs.csi.aws.com                         static-snapshot-demo   3d23h
```
3. Create a volumesnapshot in `openshift-adp` using the `volumesnapshotcontent` created in step 2. 
```
$ cat staticvolsnapshot.yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: static-snapshot-demo
  namespace: openshift-adp
spec:
  source:
    volumeSnapshotContentName: static-snapshot-content
```
```
$ oc get VolumeSnapshot static-snapshot-demo
NAME                   READYTOUSE   SOURCEPVC   SOURCESNAPSHOTCONTENT     RESTORESIZE   SNAPSHOTCLASS   SNAPSHOTCONTENT           CREATIONTIME   AGE
static-snapshot-demo   true                     static-snapshot-content   10Gi                          static-snapshot-content   6d2h           3d23h
```
4. Create a PVC and point its source to the volumesnaphot from step 3. Create a dummy pod that can mount this PVC.
```
$ cat pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mssql-pvc
spec:
  storageClassName: gp2-csi
  dataSource:
    name: static-snapshot-demo
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

```
$ cat pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: openshift-adp
spec:
  containers:
    - name: busybox
      image: quay.io/ocpmigrate/mssql-sample-app:microsoft
      command: [ "/bin/sh", "-c", "tail -f /dev/null" ]
      volumeMounts:
      - name: volume1
        mountPath: "/mnt/volume1"
  volumes:
  - name: volume1
    persistentVolumeClaim:
      claimName: mssql-pvc
  restartPolicy: Never
```
5. Run VolSync replication source CR from `openshift-adp` namespace and point the pvc value to the one 
```
$ cat rep_src.yaml
---
apiVersion: volsync.backube/v1alpha1
kind: ReplicationSource
metadata:
  name: poc1-src
  namespace: openshift-adp
spec:
  sourcePVC: mssql-pvc
  trigger:
    manual: triggertest
  restic:
    pruneIntervalDays: 15
    repository: restic-config
    retain:
      hourly: 1
      daily: 1
      weekly: 1
      monthly: 1
      yearly: 1
    copyMethod: None
```
*Note: CopyMethod is set to None, since we have already created the PVC from the transferred VolumeSnapshot*
```
oc get volumesnapshot -n mssql-persistent -o yaml
apiVersion: v1
items:
- apiVersion: snapshot.storage.k8s.io/v1
  kind: VolumeSnapshot
  metadata:
    creationTimestamp: "2022-02-23T22:10:47Z"
    finalizers:
    - snapshot.storage.kubernetes.io/volumesnapshot-as-source-protection
    generateName: velero-mssql-pvc-
    generation: 1
    labels:
      velero.io/backup-name: mssql-persistent
    name: velero-mssql-pvc-fh5f5
    namespace: mssql-persistent
    resourceVersion: "19821918"
    uid: 5b5dda14-8a71-4c2b-acf7-56249aa46311
  spec:
    source:
      persistentVolumeClaimName: mssql-pvc
    volumeSnapshotClassName: example-snapclass
  status:
    boundVolumeSnapshotContentName: snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311
    creationTime: "2022-02-23T22:10:50Z"
    readyToUse: true
    restoreSize: 10Gi
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
  
```
$ oc get volumesnapshotcontent snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311 -o yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  creationTimestamp: "2022-02-23T22:10:47Z"
  finalizers:
  - snapshot.storage.kubernetes.io/volumesnapshotcontent-bound-protection
  generation: 1
  labels:
    velero.io/backup-name: mssql-persistent
  name: snapcontent-5b5dda14-8a71-4c2b-acf7-56249aa46311
  resourceVersion: "19821935"
  uid: 9b776de5-f0c6-4ae4-8d0f-ceaf4394b874
spec:
  deletionPolicy: Retain
  driver: ebs.csi.aws.com
  source:
    volumeHandle: vol-01f9fc6449d383392
  volumeSnapshotClassName: example-snapclass
  volumeSnapshotRef:
    apiVersion: snapshot.storage.k8s.io/v1
    kind: VolumeSnapshot
    name: velero-mssql-pvc-fh5f5
    namespace: mssql-persistent
    resourceVersion: "19821379"
    uid: 5b5dda14-8a71-4c2b-acf7-56249aa46311
status:
  creationTime: 1645654250951000000
  readyToUse: true
  restoreSize: 10737418240
  snapshotHandle: snap-0f154973f29b185eb
  
  
```