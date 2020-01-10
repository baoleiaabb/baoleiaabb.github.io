---
layout:     post
title:      Split mirror using suspended IO in DB2 Universal Database
subtitle:   
date:       2020-01-10
author:     BY
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - db2

---

  

The need to maintain high availability of data, 24x7, is crucial for today's global businesses. The suspended I/O feature of IBM DB2 UDB provides an interface for online split mirroring with continuous system availability to meet this crucial need. This article defines and explains the key concepts of split mirroring and suspended I/O and provides step-by-step instructions for several different scenarios.

I'm assuming you have a basic level of familiarity with DB2 (similar to that which an IBM Certified DB2 Database Administrator might have) in order to take advantage of the technical details of this article.

## What is a split mirror?

A *split mirror* is an identical, independent, and instantaneous copy of a disk volume that can be attached to a different system. A split mirror of a DB2 UDB database refers to a set of identical, independent, and instantaneous copies of disk volumes containing all of its control and data files, which includes the entire contents of the *database directory*, all the *tablespace containers*, the *local database directory*, and the *active log directory* (if it does not reside on the database directory). The active log directory only needs to be split for creating a clone database using the `snapshot` parameter of the `db2inidb` command. The other two parameters, `standby` and `mirror`, do not require the active log directory to be split.

Accessing the split mirror does not involve copying. It is dependent on the storage vendor's implementation. Users must use the storage vendor's facilities to access the split mirror; it should not be accessed in any other way.

Following is an elaboration on the split mirror content of a DB2 UDB database:

- Database directory:

  When a database is created, information about the database, including default information, is stored in a directory hierarchy. The hierarchical directory structure is created at a location based on the information provided in the `db2 create database` command. The directory structure appears as "<database path>/<instance name>/<node name>/SQLnnnnn/." The subdirectory named *SQLnnnnn* in this structure is referred to as the *database directory*. The name of this subdirectory uses the database token and represents a database. For example, SQL00001 contains objects associated with the first database, and subsequent databases are given higher numbers: SQL00002 and so on.

  The database directory contains the database control files (SQLBP.1, SQLBP.2, SQLSPCS.1, SQLSPCS.2, SQLSGF.1, SQLSGF.2, SQLDBCON, DB2RHIST.ASC, DB2TSCHNG.HIS, SQLOGCTL.LFH, SQLOGMIR.LFH and SQLINSLK), the SQLT* subdirectories for the default SMS tablespaces required for an operational database, SQLOGDIR subdirectory for the default log path, the SYSTOOLSPACE subdirectory for the automatic statistics collection and reorganization features, and the DB2EVENT subdirectory for the default event monitors.

  The database directory corresponding to a database can easily be determined by issuing the `db2 list db directory on <database path>` command.

- Local database directory:

  A local database directory resides in each path (or "drive" for Windows) in which a database has been defined. The directory structure appears as "<database path>/<instance name>/<node name>/sqldbdir/". This directory contains one entry for each database accessible from that location. When using a split mirror solution, it is recommended to create a single database in a database path so that the local database directory ("sqldbdir") only has the entry for a single database.

- Tablespace containers:

  All tablespace containers that are not part of the database directory must be part of the split mirror. Information about tablespace containers can be found using the `db2 list tablespaces` and `db2 list tablespace containers for <tablespace-id>` commands.

- Active log directory:

  The path to log files should only be included in the split mirror for clone database. Information about the current active log directory can be found using the `db2 get db cfg for <database-alias>` command.

##### Figure 1. Split mirror contents of a DB2 UDB database

![Split mirror contents of a DB2 UDB database](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/sm_fig1.gif)

### Suspended I/O functionality

When splitting a mirror of a DB2 UDB database, it is important to ensure that there are no partial page writes occurring on the source database. One way to ensure no partial page write is to bring the database offline, a state which is absolutely not desirable in a true 24x7 production environment.

In an effort to provide continuous system availability during the split mirror process, DB2 UDB provides the ability to suspend I/O on the source database, allowing you to split the mirror while the database is still online, without requiring any downtime.

The suspended I/O functionality of DB2 UDB eliminates any partial page writes by suspending all write activities on the source database. While a database is in write suspend mode, all the tablespaces are placed into *Write Suspended* state (0x10000); however, all operations on the database function normally.

Here is a sample output of the `db2 list tablespaces` command showing the tablespace state:

##### Listing 1. Tablespace state showing write suspended

```
Tablespace ID                           = 0``Name                                    = SYSCATSPACE``Type                                    = System managed space``Contents                                = Any data``State                                   = 0x10000``Detailed explanation:``    ``Write Suspended
```

Transactions requiring disk I/O, such as flushing dirty pages from the buffer pool or flushing logs from the log buffer, may wait. However, these transactions will proceed normally once the write operations on the source database have been resumed.

A split mirror created using the suspended I/O functionality remains in "Write Suspended" state until it has been initialized by DB2 using the `db2inidb` command. The `db2inidb` command provides the options to initialize a split mirror in three different ways (clone, standby, or mirror), allowing users to meet the various objectives of its usage.

## Common ways to use split mirror

A split mirror of a DB2 UDB database created using the suspended I/O functionality provides many options to the user that make it suitable to meet various objectives. Here are some common ways that a split mirror can be used:

### Clone source database

A split mirror of a DB2 UDB database can be initialized as a snapshot of the source database, essentially an instantaneous clone of the source database. Once initialized, the split mirror becomes a fully functional database with a new log chain, it can be used to populate a test system, or it can be used for generating reports without having an impact on the production system.

### Warm standby database

A split mirror of a DB2 UDB database can also be initialized as a warm standby database, providing a quick disaster recovery option if the primary database becomes unavailable.

### Offload backup from production

DB2 UDB allows you to take a full database backup of a standby database (if it only contains DMS tablespaces) while it is in the rollforward pending state. This offloads the overhead of taking a backup on the production database. There is a special-case code in the DB2 backup utility to allow a backup after `db2inidb` with `standby` parameter. The log files produced on the source database after the mirror is split can be used for rollforward recovery. The backup of a standby database containing SMS tablespaces is currently not supported for two main reasons:

1. For taking backups of SMS tablespaces, DB2 must query the system catalogs to get a list of tables that are to be included in the backup image. On a split mirror, the catalogs will be inconsistent (until rollforward is completed). Therefore, DB2 cannot reliably get the list of tables.
2. The backup on the split mirror system will be capturing the state of the database containers on the source system at the time that the split occurred. However, the split mirror system will not have any knowledge of the database state at this time (this, too, will be resolved by the rollforward). So if there were any operations in progress on the source system that cannot be done at the same time as a backup (or equivalently, that will make the backup wait on a lock or some other synchronization primitive), the backup on the split mirror would have to be blocked the same way. But since it is not generally possible to determine whether any of these operations were in progress, backups on the split system are blocked across the board.

These restrictions do not apply to DMS containers because:

1. DB2 does not need a list of tables for taking backups of DMS tablespaces, and
2. There are DMS-specific locks that prevent incompatible operations from occurring at the same time as a backup, and these are also enforced at the time of the split.

### System-level backup and recovery

A split mirror initialized as mirror can be used as a system level backup, allowing the administrator to perform a quick file system level recovery of the database.

### High-speed backup and recovery

The suspended I/O functionality combined with the mirroring functionalities of intelligent storage servers, such as FlashCopy® technology of IBM Enterprise Storage Server® (ESS, known as Shark) or the instant split feature of EMC Symmetrix Remote Data Facility (SRDF), provide options for fast backup and recovery of a production database.

## Implementation details

The implementation of a split mirror solution consists of the following key phases:

1. [Suspending I/O on the source database](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/index.html#suspendio)
2. [Splitting the mirror of the source database](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/index.html#splitmirror)
3. [Resuming I/O on the source database](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/index.html#resumeio)
4. [Initializing the split mirror as needed](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/index.html#initializesplit)

### Suspending I/O

The I/O write operation on the source database can be suspended from the CLP (Command Line Processor) using the `db2 set write` command. The I/O can also be suspended from an application by calling the `db2SetWriteForDB` DB2 API with `DB2_DB_SUSPEND_WRITE` input value.

##### Listing 2. Suspending I/O from CLP

```
db2 connect to <``database-alias``>``db2 set write suspend for database
```

### Splitting the mirror

The splitting process is independent of DB2 UDB and is entirely dependent on the storage vendor's implementation. There are no hardware or software-specific requirements to use the suspended I/O functionality to split the mirror of a DB2 UDB database.

IBM recommends using intelligent storage servers, such as the IBM ESS or EMC Symmetrix, to establish near-instantaneous copies of the data entirely within itself. However, there are no restrictions on using the operating system commands, tools, or any other utilities to create the split mirror.

It is important to keep in mind, on a heavily used system, if the database is kept in write suspended state for an extended period of time, eventually all of the applications will hang while attempting to either flush the log buffers or the buffer pool pages to disk. No crash will occur, but the instance will simply come to a halt until I/O writes are resumed. One way of avoiding this type of situation (or at least reducing the chances of it happening) is to make the buffer pool big enough.

The database configuration parameter LOGRETAIN must be set to RECOVERY prior to splitting the mirror if it is intended to be initialized as standby or mirror database.

The steps to split a mirror are as follows:

1. Connect to the source database
2. Suspend I/O (all write activities) on the source database
3. Split the mirror (using the method of userâ€™s choice)
4. Resume I/O (all write activities) on the source database

##### Figure 2. Splitting mirror of a DB2 UDB database

![Splitting mirror of a DB2 UDB database](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/sm_fig2.gif)

### Resuming I/O

The I/O write operation on the source database can be resumed from the CLP using the `db2 set write` command. If the connection used to suspend I/O writes is lost or hung, and if all subsequent connection attempts are also hung, then I/O can be resumed using `db2 restart database` command with the `write resume` option. In this case, no crash recovery will be performed unless this command was issued after a database crash. It is important to keep in mind that the `write resume` option can only be applied to the primary database and should not be applied to a mirrored database.

The I/O on the source database can also be resumed by calling `db2SetWriteForDB` DB2 API with `DB2_DB_RESUME_WRITE` input value from an application. The DB2 API `db2DatabaseRestart` also resumes I/O (in addition to performing crash recovery) on the source database depending on the input values provided. The valid input values are `DB2_DB_SUSPEND_NONE` and `DB2_DB_RESUME_WRITE`.

```
db2 set write resume for database
```

or

```
db2 restart database <``database-alias``> write resume
```

### Initializing the split mirror

A split mirror of a DB2 UDB database created using suspended I/O remains in "Write Suspended" state until it is has been initialized into a usable state. Therefore, the split mirror of a DB2 UDB database must be initialized before it can be used.

The `db2inidb` command initializes a split mirror to a functional state by performing a crash recovery or by placing it into a rollforward pending state. The `db2inidb` command can only be used to initialize a split mirror created using suspended I/O.

Depending on the parameter (snapshot, standby, or mirror) used in the `db2inidb` command, a split mirror can be initialized as:

- Clone of the source database
- Warm standby database
- Mirror database

In a partitioned environment each mirror partition must be initialized before using the database. Each mirror partition can be initialized independently, or `db2_all` can be used to run it on all mirror partitions simultaneously.

During the initialization of the split mirror it is possible to even relocate the database in terms of the database name, database directory path, container path, log path, and the instance name associated with the database.

To relocate the database, the `db2inidb` command must be issued with the `relocate` parameter along with a configuration file that contains the original and the intended configuration information. The `relocate` parameter invokes the `db2relocatedb` command in the background and thus allows initialization of the split mirror on the same system where the source database resides.

The format of the relocate configuration file (text file) is as follows:

##### Listing 3. Relocate configuration file

```
DB_NAME=oldName,newName``DB_PATH=oldPath,newPath``INSTANCE=oldInst,newInst``NODENUM=nodeNumber``LOG_DIR=oldDirPath,newDirPath``CONT_PATH=oldContPath1,newContPath1``CONT_PATH=oldContPath2,newContPath2
```

### Use split mirror as a clone database

A split mirror of a DB2 UDB database created using the suspended I/O functionality can be used as a clone of the source database by initializing it with the `snapshot` parameter of the `db2inidb` command.

The initialization of the split mirror as snapshot performs a crash recovery, rolls back all uncommitted transactions, and makes the database available for any read or write operation. It is essential to have all the log files from the source database that were active at the time of the split to complete the crash recovery. The active log directory of the split mirror database must not contain any log file that is not a part of the split mirror.

After the completion of the crash recovery, a new log chain will be started. Therefore, rollforward through any of the logs from the source database is not possible.

A database backup taken on the clone database can be restored on the source database. However, any logs from the source database generated after the split cannot be applied. Therefore, it will be a version level copy only.

Once the `db2inidb` command with the `snapshot` parameter is issued against a split mirror, whether it succeeds or fails, the `standby` or `mirror` parameter cannot be used as it starts a new log chain.

Here are the steps required on the target system to use a split mirror as a clone database:

1. Create the database instance (if doesnâ€™t exist).

2. Catalog the database (system database directory) with ON parameter:

   

3. Make the split mirror accessible:

   1. Mount the split mirror copy of the database directory.
   2. Mount the split mirror copy of all tablespace containers. (If the containers are located in several directories, then all container directories must be mounted.)
   3. Mount the local database directory (sqldbdir).
   4. Mount the split mirror copy of the log directory (if the log files are located in a directory other than the database directory on the source database).

4. Start the database manager (ensure that the DB2 registry variable DB2INSTANCE is set):

   

5. Initialize the split mirror:

   

##### Figure 3. Create a clone database using split mirror

![Create a clone database using split mirror](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/sm_fig3.gif)

### Use split mirror as a standby database

A split mirror of a DB2 UDB database created using the suspended I/O functionality can be used as a standby database by initializing it with the `standby` parameter of the `db2inidb` command.

After the initialization, the split mirror is placed into a rollforward pending state as a standby database. The database will be kept in a continuous rollforward pending state until the rollforward has been stopped.

In order to keep the standby database as current as possible, new inactive log files from the source database should be continually applied on the standby database as they become available. A userexit program can be used to automate the continuous retrieval of the inactive log files from the source database. If a userexit is used, both the source and the target databases must be configured with the same userexit program.

Here are the steps required on the target system to use a split mirror as a standby database:

1. Create the database instance (if doesnâ€™t exist).

2. Catalog the database (system database directory) with ON parameter:

   

3. Make the split mirror accessible:

   1. Mount the split mirror copy of the database directory.
   2. Mount the split mirror copy of all tablespace containers. (If the containers are located in several directories, then all container directories must be mounted.)
   3. Mount the local database directory (sqldbdir).

4. Start the database manager (ensure that the DB2 registry variable DB2INSTANCE is set):

   

5. Initialize the split mirror:

   

6. Continually copy over or retrieve the inactive log files from the source database and roll them forward on the target database:

   

If the source database becomes unavailable (for example, if it crashes), the standby database on the target system can be activated for user access. In order to activate the standby database, a user must take the standby database out of the rollforward pending state. This can be done by issuing the `db2 rollforward` command with the `stop` or `complete` parameter to bring the database into a consistent state. Once the database is in a consistent state, users can switch over to the standby database to continue their work. The user applications will have to make new connections to this activated standby database. The log files generated on the standby database cannot be applied on the source database.

##### Figure 4. Create a standby database using split mirror

![Create a standby database using split mirror](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/sm_fig4.gif)

### Use split mirror as a mirror database

A split mirror created using suspended I/O can be used as a mirror database for quick system level recovery by copying it over the source database and then initializing it using the `db2inidb` command with the `mirror` parameter.

After the initialization of the split mirror as mirror, the database will be placed into a rollforward pending state. The existing log files will be used for rollforward recovery on the source database. No crash recovery is initiated, and the database remains inconsistent until a rollforward to the end of logs and stop/complete has been issued. It is important to note that the split mirror must remain in the write suspended state until it is copied on top of the source database.

Here are the actions required on the source system to set up the mirror database:

1. Shut down the source database instance:

   

2. Copy the split mirror on top of the source database. Replace the source database (excluding the active log directory) with the mirror copy of the database.

3. Start the source database instance:

   

4. Initialize the mirror copy on the source database:

   

5. Rollforward to end of logs and stop:

   

##### Figure 5. Create a mirror database using split mirror

![Create a mirror database using split mirror](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/sm_fig5.gif)

### Handling partitioned environments

In a multi-partitioned environment, every partition is treated as a separate database. Therefore, I/O writes must be suspended on each partition during the split mirror process. Similarly, the I/O writes must be resumed on each partition after the mirror is split. The same applies when initializing the split mirror using the `db2inidb` command, which must be run on each mirrored partition before using the database.

Because each partition is treated as a separate database, I/O writes can be suspended on each data partition independently of all others. Therefore, a user does not need to issue a `db2_all` to suspend I/O on all of the partitions. However, if each partition is suspended independently, the I/O write on the catalog partition must be suspended last. In a multi-partitioned database, an attempt to suspend I/O on any of the non-catalog partitions will require a connection to the catalog node for authorization. If the catalog partition is suspended, then the connection attempt may hang.

In a partitioned database environment, the recommendation is to suspend I/O writes on all partitions independently, suspending the catalog partition last. This means that I/O can be suspended without affecting the other partitions.

### Example

If a partitioned database on AIX has four database partitions (0,1,2,3), where the catalog partition is 0, then you can suspend and resume I/O as follows:

##### Listing 4. Commands to suspend and resume I/O on AIX

```
db2 export DB2NODE=1``db2 terminate``db2 connect to <``database-alias``>``db2 set write suspend for database` `db2 export DB2NODE=2``db2 terminate``db2 connect to <``database-alias``> ``db2 set write suspend for database` `db2 export DB2NODE=3``db2 terminate``db2 connect to <``database-alias``>``db2 set write suspend for database` `db2 export DB2NODE=0``db2 terminate``db2 connect to <``database-alias``>``db2 set write suspend for database` `db2_all restart database <``database-alias``> set write resume
```

When initializing the split mirror, the `db2inidb` command does not require any connections to the database. Therefore, this command can be run independently on each partition, or `db2_all` can be issued to run `db2inidb` on all partitions simultaneously. The only requirement is for the database manager to be started.

If there are some in doubt transactions at the time of the split of the mirror (of a partitioned database) on some partitions, the `db2inidb` with the `snapshot` parameter may fail during the sideway recovery. This is an expected behavior. In order to resolve an in doubt transaction, a data partition may need to contact another data partition. If the data partition which is being contacted has not yet had the `db2inidb` run, the request will be rejected and therefore will fail with one of the following error messages:

- SQL20153N The database's split image is in the suspended state.
- SQL1391N The database is already in use by another instance of the database manager.

SQL20153N would only occur if `db2inidb` is issued sequentially on each data partition, whereas SQL1391N can occur when `db2inidb` is run in parallel across all data partitions.

In this case, just re-issuing the restart database command on the failed partitions will resolve the problem.

So, the recommendation for initializing a split mirror of a DB2 UDB database in a multi-partitioned database would be:

1. Issue `db2inidb` on all data partitions simultaneously using `db2_all`, or issue `db2inidb` on each partition individually in a serial manner.
2. Issue a `db2 restart database` command (without `write resume` option) on all partitions to ensure initiation of sideway recovery to resolve any in doubt transactions (as it may not be convenient or easy for users to look for a sideway recovery failure).

Please note, the above is applicable only when the `snapshot` parameter is used in the `db2inidb` command. As `standby` and `mirror` options do not perform crash recovery, this does not apply to them.

## Debugging split mirror problems

In a split mirror scenario, there are always three sets of actions involved:

1. Suspending and resuming I/O on the source database using the `db2 set write` command.
2. Splitting of the mirror.
3. Initializing the split mirror using the `db2inidb` command.

When debugging a problem related to suspended I/O, the IBM DB2 technical support team will investigate actions 1 and 3. Since the splitting process takes place completely outside of DB2 UDB, the users should engage the appropriate storage vendor support to debug any issues surrounding the splitting process they use.

The general set of data collected and provided to IBM Support should include:

- Copies of db2diag.log, output of `db2level`command, database manager configuration (`db2 get dbm cfg`), and database configuration (`db2 get db cfg for <database-alias>`) from both the source and the target systems
- Copy of SQLOGCTL.LFH file (from the source database) immediately after the `db2 set write` command is issued
- Copies of SQLOGCTL.LFH file from the target system before and after issuing the `db2inidb` command
- The complete details of the exact method and third party commands used (in sequence) to split the mirror

## Summary

In this article, you have learned the concept of split mirror in context of a DB2 UDB database and how the suspended I/O functionality can be used to implement various high availability solutions to meet the demanding needs of continuous system availability. You have also discovered the detailed procedures on how to suspend/resume I/O, split a mirror, and initialize it to make it usable on both partitioned and non-partitioned environment.

### Disclaimer

The information provided in this document is distributed on "as is" basis without any warranty, either expressed or implied, and neither IBM nor the author is responsible for any consequences of its use.



#### Downloadable resources

- [PDF of this content](https://www.ibm.com/developerworks/data/library/techarticle/dm-0508quazi/dm-0508quazi-pdf.pdf)



#### Related topics

- [*IBM DB2 Universal Database: Data Recovery and High Availability Guide and Reference Version 8.2*](ftp://ftp.software.ibm.com/ps/products/db2/info/vr82/pdf/en_US/db2hae81.pdf) provides detailed information about, and shows you how to use, the IBM DB2 Universal Database (UDB) backup, restore, and recovery utilities. The book also explains the importance of high availability and describes DB2 failover support on several platforms.
- [*IBM DB2 Universal Database: Command Reference Version 8.2*](ftp://ftp.software.ibm.com/ps/products/db2/info/vr82/pdf/en_US/db2n0e81.pdf) provides information about the use of system commands and the IBM DB2 Universal Database command line processor (CLP) to execute database administrative functions.
- [*IBM DB2 Universal Database: Administrative API Reference Version 8.2*](ftp://ftp.software.ibm.com/ps/products/db2/info/vr82/pdf/en_US/db2b0e81.pdf) provides information about the use of application programming interfaces (APIs) to execute database administrative functions.
- "[Split Mirror using Suspended I/O in IBM DB2 Universal Database Version 7](http://www.ibm.com/developerworks/db2/library/techarticle/0204quazi/0204quazi.html)" (developerWorks, April 2002) provides an overview of suspended I/O feature of DB2 UDB in V7.2.

 

