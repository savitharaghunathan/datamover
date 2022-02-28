# Data Mover


## Description

Data Mover Plugin for OADP enables the backup and restore of CSI volumes by making them portable leveraging VolSync's Restic data mover.

## Background

Velero CSI plugin is currently used for backing up and restoring CSI snapshots for valid storage providers. The current limitation with CSI snapshots are that they might get deleted when there is a cluster failure. To ensure that the snapshots are available in case of a disaster, we need a `Data Mover` that can backup to/restore CSI snaphots from a user defined object storage. 

## Roadmap

We are envisioning to ship the phase 1 of plugin approach as a tech preview feature for OADP 1.1. We are aware of the gaps in plugin approach and will be working towards adding a vendor agnostic data mover to Velero upstream. 

## Goals
* Support for backing up CSI snapshots to a user defined backup location
* Allow portability of the CSI snapshots
* Allow user to opt-in/out for backing up PVCs with the data mover
* Allow user to configure restic repository
* Allow 3rd party data mover plugins
* Guidelines for writing your own data mover plugin

## Non-Goals
* Support for multiple restic repositories in a single backup
* Support for Data Mover plugins other than Volsync restic data mover

## Design Proposal

### Basic workflow
1. Annotate PVC with Data Mover set to true (Will be used only in opt-out method)
2. Create a backup/restore object
3. Velero backup/restore process triggers VolSync backup/restore
4. Velero process completes 
5. Update status

##### Backup
1. Velero backup request is created
2. Data Mover plugin creates a volsync `ReplicationSource` CR for every PVC that's backed up 
3. Data Mover plugin waits for `ReplicationSource` backup to complete
4. Velero's backup request continues and is processed 

##### Restore
1. Velero restore request is created
2. Data Mover plugin creates empty PVC's with a `ReplicationDestination` CR to restore volume data, and waits for process to complete
3. Data Mover plugin waits for `ReplicationDestination` restore to complete 
4. Velero's restore request continues and is processed

### Preferred design - Plugin approach

[Link to flowchart](https://lucid.app/lucidchart/fe070c7b-f457-4ea9-a113-ca4bbc308328/edit?invitationId=inv_dde0276f-3e6b-4ff4-9050-e1936e6cd67f) - Refer to plugin approach page.

VolSync will be used as the Data Mover in this design. For now, VolSync is the only plugin that will be supported and `restic` will be the supported method for backup & restore of PVCs. Restic repository details are configured in a `secret` object which gets used by the VolSync's resources. This design takes advantage of VolSync's two resources - `ReplicationSource` & `ReplicationDestination`. `ReplicationSource` object helps with taking a backup of the PVCs and using restic to move it to the storage specified in the restic secret. `ReplicationDestination` object takes care of restoring the backup from the restic repository. There will be a 1:1 relationship between the replication src/dest CRs and PVCs.

There will be a section in DPA CR that will allow the user to specify the details of data mover configuration. 


#### Phase 1: Blanket method

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


Like any other plugins, DataMover plugin gets deployed as an init container in the velero deployment. Enabling this config `DataMoverEnable: True/False` in the DPA CR, will bypass velero CSI snapshots and Volsync will be used to for all the PVC backup and restore.

When a velero backup gets triggered, velero takes backup of the resources. When PVCs are encountered, velero pauses and triggers DM plugin to take backup of the PVCs. Once that completes, velero resumes backing up resources including the `ReplicationSource` objects.

When a Velero restore gets triggered, the data mover plugin operates on `ReplicationSource` resources and creates empty PVCs with names matching the ones in the file. Then it creates a `ReplicationDestination`  object to restore the PVCs from the given CSI snapshots. Once this process completes, Velero proceeds with restoring the rest of the resources.

A status controller is created to watch VolSync CRs. It watches the `ReplicationSource` and`ReplicationDestination` objects and updates Backup/Restore CR events. 

#### Phase 2: opt-out method

```
...
DataMoverEnable: True/False
DMconfig:
  - Repo: <path to restic repo>
    Password: <encryption password>
    Secret: <credentials for restic repo access>
...
```

User will provide the restic repo details. This may be an encryption password and BSL reference, and DPA controller will create restic secret that uses BSL info, or they can supply their own backup target and access credentials. Once a backup request is created, velero takes backup of the resources. When it encounters a PVC, the following annotation is checked

`oadp.datamover.enabled=False`

If the annotation is present then the PVC will be opted out of thebackup. PVCs without the above annotation will be backed up by VolSync restic data mover and the velero backup process continues. 


![Plugin Approach flowchart](https://i.imgur.com/hn48DNf.jpg)

#### Blocking vs Non-blocking

This design will follow non-blocking approach while performing a backup/restore. Status controller will be watching the data mover CRs and add events to the backup/restore CR accordingly.

##### Assumptions:
* Velero backup & DM plugin backup can happen simultaneously. Similarly for restore. There is no order of operations
* Need for a new API hasnt been identified
* Only one restic repo is configured
* When the plugin is enabled, the default assumption is that the user wants to backup/restore CSI snapshots using data mover plugin.

##### Limitations:
1. Adding annotations to PVC can lead to user error. 

### Alternate design - OADP Controller approach 
[Link to flowchart](https://lucid.app/lucidchart/fe070c7b-f457-4ea9-a113-ca4bbc308328/edit?invitationId=inv_dde0276f-3e6b-4ff4-9050-e1936e6cd67f) - Refer to controller approach page ** _Feel free to edit_**

1. Similar to Scott's design
2. Custom DM controller that watches on Velero backup and triggers volsync to backup to cold storage and completes velero backups.
3. Custom DM controller that watches Velero restore, triggers VolSync restore and then restores velero backups

#### Limitations:
1. Only supports VolSync for now
2. Plugin interface + implementation arch is reinventing the wheel and complicated
3. Maintainence of plugins


### For Inspiration A.K.A Existing work
* [Scott's Data Mover design](https://github.com/konveyor/data-mover/blob/master/docs/design/initial-design.md)
* [Velero's Data Mover design proposal](https://github.com/vmware-tanzu/velero/pull/4461)
    * <i>Note: This PR has been closed since the developer is leaving the team.</i>



---


## Discussion

Refer to https://hackmd.io/@t3J7ofpkTWq13capOOepdQ/SJ9VtlsRF 

### Restic usage thoughts with volsync

Today, volsync expects `ReplicationSource` custom resources to be created in the namespace where the application's PVC exists. This means that the DM plugin should be creating the volsync CRs in the namespace where the PVCs will live. It may be desired we change volsync to work where `ReplicationSource` gets created in `openshift-adp` and volsync controller backs up PVCs in another namespace. 

The other reason this is important is because the restic secret that `ReplicationSource` relies on currently must exist in the application namespace. This would mean the user has to set this secret up prior to backup, or the OADP operator will be responsible for replicating this secret into the app namespaces. It would be ideal for this to exist in openshift-adp and used for all PVC backups, meaning that the secret can be referenced cross-namespace for a given `ReplicationSource`.

This secret is something the user must supply some manual input when data mover is enabled. User can either supply just an encryption password and BSL reference, and we will create restic secret that uses BSL info, or they can supply their own backup target and access credentials.

### Identified Gaps
- The Velero plugin does not wait for a snapshot to complete
- Blocking vs Non-blocking: recovery times
- Watching and reporting the VolSync process
- Handling the DPA config if this is a tech preview
- VolSync restic secret needs to be replicated. Replication source does not have a way to cross reference the secret across namespace

## Misc. details
Links to learnings & background research - 
https://hackmd.io/@t3J7ofpkTWq13capOOepdQ/SJ9VtlsRF


## open questions
Restic secret has the info regarding restric repo. Do we have to create a new restic secret with a new repo for every VolSync CR?  