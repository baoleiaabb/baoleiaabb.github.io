---
layout:     post
title:      Backup HANA DB using storage snapshot
subtitle:   
date:       2020-01-10
author:     BY
header-img: img/post-bg-map.jpg
catalog: true
tags:
    - hana
 
---

​      This POC will demonstrate the ability to use storage snapshots as a backup of the HANA database. I'll take a LVM snapshot, imitating a snapshot of the storage. Then I will try to restore the database using the resulting snapshot. The required snapshot have be taken only for the "data" area, but it can also contain a "log" and "shared". The POC was performed on a test, almost inactive system and therefore must be repeated on the system under load.

This POC was made for HANA1; the procedure for HANA2 is slightly different, because it works in multitenant mode. Using a HANA2 snapshot as a backup is supported for one tenant instance.

Pros:

- Space efficiency. The snapshot will take storage capacity only for changes, still be FULL backup of DB.
- Fast. Takes seconds for a database of any size.
- There is no waste of server resources. Data does not leave storage, there is no traffic or IO.
- Fast recovery. Just stop the database, revert to snapshot and start the database.

Contras:

- No data copied to another location for disaster recovery.
- It is hard to use as recovery on other system

### 1  Taking BACKUP

Let's add some data to the database to check the recovery proces

```
ndbadm@hana-snap:/> hdbsql -u SYSTEM -i 00
Password: 
 ..
hdbsql NDB=> CREATE TABLE t1(a INT, b INT, c INT);
0 rows affected (overall time 3576 usec; server time 1893 usec)

hdbsql NDB=> INSERT INTO t1 VALUES(1,12,13);
1 row affected (overall time 2672 usec; server time 358 usec)
```

Instruct HANA to start backup using a storage snapshot, and get resulting `BACKUP_ID`:

```
hdbsql NDB=> BACKUP DATA CREATE SNAPSHOT;
0 rows affected (overall time 133.962 msec; server time 132.589 msec)

hdbsql NDB=> select BACKUP_ID from SYS.M_BACKUP_CATALOG where ENTRY_TYPE_NAME='data snapshot' and STATE_NAME='prepared';
1 row selected (overall time 12.607 msec; server time 1765 usec)

BACKUP_ID
1558619489868
lines 1-2/2 (END)
```

Database files becomes in a consistent state; result is similar to the Oracle "begin backup" command. The last command returns a `BACKUP_ID` that will be used to complete the backup and resume normal database operations.

As a result of executing the `CREATE SNAPSHOT` command, HANA creates an additional file that includes the database metadata:

```
root@hana-snap:~ # find /hana/data/ -name "snap*"
/hana/data/NDB/mnt00001/hdb00001/snapshot_databackup_0_1
```

Now we need to take a snapshot of logical volume. You must have some free space in your PV to be able create a snapshot. The free space must be more than the expected LV changes over the snapshot lifetime. It is possible to autoextend snapshot size using parameters snapshot_autoextend_threshold and snapshot_autoextend_percent at `/etc/lvm/lvm.conf`.

```
root@hana-snap:~ # pvs
  PV         VG     Fmt  Attr PSize   PFree
  /dev/sdb   rootvg lvm2 a--   16.00g   4.00g
  /dev/sdc   hanavg lvm2 a--  250.00g 145.00g
```

Use the `sync` command to flush disk buffers. This is important to ensure that the storage snapshot will be in a consistent state.

```
root@hana-snap:~ # sync; sync; sync
```

And finally create a snapshot:

```
root@hana-snap:~ # lvcreate -s -n data_backup_20190523 -l 10%FREE /dev/hanavg/data
  device-mapper: remove ioctl on (254:7) failed: Device or resource busy
  Logical volume "data_backup_20190523" created.
```

Now we should finish backup (similar to the Oracle's "end backup" command)

```
hdbsql NDB=> BACKUP DATA CLOSE SNAPSHOT BACKUP_ID 1558619489868 SUCCESSFUL 'data_backup_20190523';
0 rows affected (overall time 385.929 msec; server time 383.554 msec)
```

Where BACKUP_ID is what we got at the beginning of the backup, and the last string is a description that helps us locate the snapshot.

Thats it.

## 2 Restoring HANA database using snapshot

Lets insert some other data to be sure we have restored version:

```
hdbsql NDB=> CREATE TABLE t2(a INT, b INT, c INT);
0 rows affected (overall time 7028 usec; server time 4428 usec)

hdbsql NDB=> INSERT INTO t2 VALUES(2,120,130);
1 row affected (overall time 6651 usec; server time 790 usec)

hdbsql NDB=> exit
ndbadm@hana-snap:/> HDB stop
 ..
 OK
```

Now, unmount the data volume, revert to snapshot and mount it back:

```
root@hana-snap:~ # umount /hana/data
root@hana-snap:~ # lvconvert --merge /dev/hanavg/data_backup_20190523
  Merging of volume data_backup_20190523 started.
  data: Merged: 99.7%
  data: Merged: 100.0%
  Merge of snapshot into logical volume data has finished.
  Logical volume "data_backup_20190523" successfully removed
root@hana-snap:~ # mount /hana/data
```

Create recovery command file and start an instance:

```
ndbadm@hana-snap:/> echo "RECOVER DATA USING SNAPSHOT CLEAR LOG" > $DIR_INSTANCE/work/recoverInstance.sql
ndbadm@hana-snap:/> HDB start
```

An HDB was started, all expected data were in place, i.e. table t1 was there and t2 wasn't.