---
layout:     post
title:      oracle使用bbed跳过归档
subtitle:   
date:       2020-01-10
author:     BY
header-img: img/post-bg-universe.jpg
catalog: true
tags:
    - bbed

---

   当数据文件需要recover时候,缺少归档,但数据库未开归档这时候如何处理? 通常这个时候我们想立即打开数据库,并允许丢失部分数据该如何处理?

​     获取数据库中所有数据文件:

select file#,name from v$datafile

```
 FILE# NAME
---------- --------------------------------------------------
         1 /oracle/app/oracle/oradata/TEST2/system01.dbf
         2 /oracle/app/oracle/oradata/TEST2/sysaux01.dbf
         3 /oracle/app/oracle/oradata/TEST2/undotbs01.dbf
         4 /oracle/app/oracle/oradata/TEST2/users01.dbf
         5 /oracle/app/oracle/oradata/TEST2/TS1_7.xtf
         6 /oracle/app/oracle/oradata/TEST2/TS2_8.xtf
         7 /oracle/app/oracle/oradata/TEST2/test_01.dbf
```

instance crash 时发现需要recover datafile,而1_648_928161908.dbf却不存在:

```\
SQL> alter session set nls_date_format='YYYY-MM-DD HH24:MI:SS';

Session altered.

SQL> recover datafile  7 ;
ORA-00279: change 3290730 generated at 01/03/2017 09:38:38 needed for thread 1
ORA-00289: suggestion : /oracle/app/oracle/archlog/1_648_928161908.dbf
ORA-00280: change 3290730 for thread 1 is in sequence #648

Specify log: {<RET>=suggested | filename | AUTO | CANCEL}
CANCEL
Media recovery cancelled.
```

通过bbed查看需要恢复的数据文件,如下所示下面3个值需要修改:

```
oracle@mysqldb01:/home/oracle$ bbed filename=/oracle/app/oracle/oradata/TEST2/test_01.dbf blocksize=8192 password=blockedit

BBED: Release 2.0.0.0.0 - Limited Production on Tue Jan 3 10:00:30 2017

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! ***************

BBED> set block 1
        BLOCK#          1

BBED>  map
 File: /oracle/app/oracle/oradata/TEST2/test_01.dbf (0)
 Block: 1                                     Dba:0x00000000
------------------------------------------------------------
 Data File Header

 struct kcvfh, 860 bytes                    @0       

 ub4 tailchk                                @8188    


BBED> p kcvfh
struct kcvfh, 860 bytes                     @0       
   struct kcvfhbfh, 20 bytes                @0       
      ub1 type_kcbh                         @0        0x0b
      ub1 frmt_kcbh                         @1        0xa2
      ub1 spare1_kcbh                       @2        0x00
      ub1 spare2_kcbh                       @3        0x00
      ub4 rdba_kcbh                         @4        0x00000001
      ub4 bas_kcbh                          @8        0x00000000
      ub2 wrp_kcbh                          @12       0x0000
      ub1 seq_kcbh                          @14       0x01
      ub1 flg_kcbh                          @15       0x04 (KCBHFCKV)
      ub2 chkval_kcbh                       @16       0x8331
      ub2 spare3_kcbh                       @18       0x0000
   struct kcvfhhdr, 76 bytes                @20      
      ub4 kccfhswv                          @20       0x00000000
      ub4 kccfhcvn                          @24       0x0b200400
      ub4 kccfhdbi                          @28       0x3c549ef2
      text kccfhdbn[0]                      @32      T
      text kccfhdbn[1]                      @33      E
      text kccfhdbn[2]                      @34      S
      text kccfhdbn[3]                      @35      T
      text kccfhdbn[4]                      @36      2
      text kccfhdbn[5]                      @37       
      text kccfhdbn[6]                      @38       
      text kccfhdbn[7]                      @39       
      ub4 kccfhcsq                          @40       0x00009a8c
      ub4 kccfhfsz                          @44       0x000a0000
      s_blkz kccfhbsz                       @48       0x00
      ub2 kccfhfno                          @52       0x0007
      ub2 kccfhtyp                          @54       0x0003
      ub4 kccfhacid                         @56       0x00000000
      ub4 kccfhcks                          @60       0x00000000
      text kccfhtag[0]                      @64       
      text kccfhtag[1]                      @65       
      text kccfhtag[2]                      @66       
      text kccfhtag[3]                      @67       
      text kccfhtag[4]                      @68       
      text kccfhtag[5]                      @69       
      text kccfhtag[6]                      @70       
      text kccfhtag[7]                      @71       
      text kccfhtag[8]                      @72       
      text kccfhtag[9]                      @73       
      text kccfhtag[10]                     @74       
      text kccfhtag[11]                     @75       
      text kccfhtag[12]                     @76       
      text kccfhtag[13]                     @77       
      text kccfhtag[14]                     @78       
      text kccfhtag[15]                     @79       
      text kccfhtag[16]                     @80       
      text kccfhtag[17]                     @81       
      text kccfhtag[18]                     @82       
      text kccfhtag[19]                     @83       
      text kccfhtag[20]                     @84       
      text kccfhtag[21]                     @85       
      text kccfhtag[22]                     @86       
      text kccfhtag[23]                     @87       
      text kccfhtag[24]                     @88       
      text kccfhtag[25]                     @89       
      text kccfhtag[26]                     @90       
      text kccfhtag[27]                     @91       
      text kccfhtag[28]                     @92       
      text kccfhtag[29]                     @93       
      text kccfhtag[30]                     @94       
      text kccfhtag[31]                     @95       
   ub4 kcvfhrdb                             @96       0x00000000
   struct kcvfhcrs, 8 bytes                 @100     
      ub4 kscnbas                           @100      0x0029a4d3
      ub2 kscnwrp                           @104      0x0000
   ub4 kcvfhcrt                             @108      0x37822685
   ub4 kcvfhrlc                             @112      0x3752a074
   struct kcvfhrls, 8 bytes                 @116     
      ub4 kscnbas                           @116      0x000e2006
      ub2 kscnwrp                           @120      0x0000
   ub4 kcvfhbti                             @124      0x00000000
   struct kcvfhbsc, 8 bytes                 @128     
      ub4 kscnbas                           @128      0x00000000
      ub2 kscnwrp                           @132      0x0000
   ub2 kcvfhbth                             @136      0x0000
   ub2 kcvfhsta                             @138      0x0000 (NONE)
   struct kcvfhckp, 36 bytes                @484               --1 修改  修改scn base
      struct kcvcpscn, 8 bytes              @484     
         ub4 kscnbas                        @484      0x0032366a
         ub2 kscnwrp                        @488      0x0000
      ub4 kcvcptim                          @492      0x3791a09e   --2 修改 checkpoint time 
      ub2 kcvcpthr                          @496      0x0001
      union u, 12 bytes                     @500     
         struct kcvcprba, 12 bytes          @500          ---3
            ub4 kcrbaseq                    @500      0x00000288   --十进制表示648 media recover 需要的 redo sequence    
            ub4 kcrbabno                    @504      0x000049e8
            ub2 kcrbabof                    @508      0x0010
      ub1 kcvcpetb[0]                       @512      0x02
      ub1 kcvcpetb[1]                       @513      0x00
      ub1 kcvcpetb[2]                       @514      0x00
      ub1 kcvcpetb[3]                       @515      0x00
      ub1 kcvcpetb[4]                       @516      0x00
      ub1 kcvcpetb[5]                       @517      0x00
      ub1 kcvcpetb[6]                       @518      0x00
      ub1 kcvcpetb[7]                       @519      0x00
   ub4 kcvfhcpc                             @140      0x00000013
   ub4 kcvfhrts                             @144      0x3791a514
   ub4 kcvfhccc                             @148      0x00000012
   struct kcvfhbcp, 36 bytes                @152     
      struct kcvcpscn, 8 bytes              @152     
         ub4 kscnbas                        @152      0x00000000
         ub2 kscnwrp                        @156      0x0000
      ub4 kcvcptim                          @160      0x00000000
      ub2 kcvcpthr                          @164      0x0000
      union u, 12 bytes                     @168     
         struct kcvcprba, 12 bytes          @168     
            ub4 kcrbaseq                    @168      0x00000000
            ub4 kcrbabno                    @172      0x00000000
            ub2 kcrbabof                    @176      0x0000
      ub1 kcvcpetb[0]                       @180      0x00
      ub1 kcvcpetb[1]                       @181      0x00
      ub1 kcvcpetb[2]                       @182      0x00
      ub1 kcvcpetb[3]                       @183      0x00
      ub1 kcvcpetb[4]                       @184      0x00
      ub1 kcvcpetb[5]                       @185      0x00
      ub1 kcvcpetb[6]                       @186      0x00
      ub1 kcvcpetb[7]                       @187      0x00
   ub4 kcvfhbhz                             @312      0x00000000
   struct kcvfhxcd, 16 bytes                @316     
      ub4 space_kcvmxcd[0]                  @316      0x00000000
      ub4 space_kcvmxcd[1]                  @320      0x00000000
      ub4 space_kcvmxcd[2]                  @324      0x00000000
      ub4 space_kcvmxcd[3]                  @328      0x00000000
   sword kcvfhtsn                           @332      8
   ub2 kcvfhtln                             @336      0x0007
   text kcvfhtnm[0]                         @338     T
   text kcvfhtnm[1]                         @339     E
   text kcvfhtnm[2]                         @340     S
   text kcvfhtnm[3]                         @341     T
   text kcvfhtnm[4]                         @342     _
   text kcvfhtnm[5]                         @343     0
   text kcvfhtnm[6]                         @344     1
   text kcvfhtnm[7]                         @345      
   text kcvfhtnm[8]                         @346      
   text kcvfhtnm[9]                         @347      
   text kcvfhtnm[10]                        @348      
   text kcvfhtnm[11]                        @349      
   text kcvfhtnm[12]                        @350      
   text kcvfhtnm[13]                        @351      
   text kcvfhtnm[14]                        @352      
   text kcvfhtnm[15]                        @353      
   text kcvfhtnm[16]                        @354      
   text kcvfhtnm[17]                        @355      
   text kcvfhtnm[18]                        @356      
   text kcvfhtnm[19]                        @357      
   text kcvfhtnm[20]                        @358      
   text kcvfhtnm[21]                        @359      
   text kcvfhtnm[22]                        @360      
   text kcvfhtnm[23]                        @361      
   text kcvfhtnm[24]                        @362      
   text kcvfhtnm[25]                        @363      
   text kcvfhtnm[26]                        @364      
   text kcvfhtnm[27]                        @365      
   text kcvfhtnm[28]                        @366      
   text kcvfhtnm[29]                        @367      
   ub4 kcvfhrfn                             @368      0x00000400
   struct kcvfhrfs, 8 bytes                 @372     
      ub4 kscnbas                           @372      0x00000000
      ub2 kscnwrp                           @376      0x0000
   ub4 kcvfhrft                             @380      0x00000000
   struct kcvfhafs, 8 bytes                 @384     
      ub4 kscnbas                           @384      0x00000000
      ub2 kscnwrp                           @388      0x0000
   ub4 kcvfhbbc                             @392      0x00000000
   ub4 kcvfhncb                             @396      0x00000000
   ub4 kcvfhmcb                             @400      0x00000000
   ub4 kcvfhlcb                             @404      0x00000000
   ub4 kcvfhbcs                             @408      0x00000000
   ub2 kcvfhofb                             @412      0x0000
   ub2 kcvfhnfb                             @414      0x0000
   ub4 kcvfhprc                             @416      0x3121c97a
   struct kcvfhprs, 8 bytes                 @420     
      ub4 kscnbas                           @420      0x00000001
      ub2 kscnwrp                           @424      0x0000
   struct kcvfhprfs, 8 bytes                @428     
      ub4 kscnbas                           @428      0x00000000
      ub2 kscnwrp                           @432      0x0000
   ub4 kcvfhtrt                             @444      0x00000000
```

先查看正确数据文件:

```
oracle@mysqldb01:/home/oracle$  bbed filename=/oracle/app/oracle/oradata/TEST2/TS2_8.xtf  blocksize=8192 password=blockedit

BBED: Release 2.0.0.0.0 - Limited Production on Tue Jan 3 10:04:55 2017

Copyright (c) 1982, 2011, Oracle and/or its affiliates.  All rights reserved.

************* !!! For Oracle Internal Use only !!! ***************

BBED> set block 1 
        BLOCK#          1

BBED> map
 File: /oracle/app/oracle/oradata/TEST2/TS2_8.xtf (0)
 Block: 1                                     Dba:0x00000000
------------------------------------------------------------
 Data File Header

 struct kcvfh, 860 bytes                    @0       

 ub4 tailchk                                @8188    

BBED> p kcvfh.kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x00323a13
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x3791a2bd
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x00000292
         ub4 kcrbabno                       @504      0x0000021f
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00
```

由于linux是 little endians,3个值修改如下

```
 m /x 133a offset 484
 m /x bda2 offset 492
 m /x 9202 offset 500  
```

```
BBED>   m /x 133a offset 484
 File: /oracle/app/oracle/oradata/TEST2/test_01.dbf (0)
 Block: 1                Offsets:  484 to  995           Dba:0x00000000
------------------------------------------------------------------------
 133a3200 00000000 9ea09137 01000000 88020000 e8490000 10004b8f 02000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 0d000d00 0d000100 00000000 00000000 00000000 02000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED>  m /x bda2 offset 492
 File: /oracle/app/oracle/oradata/TEST2/test_01.dbf (0)
 Block: 1                Offsets:  492 to 1003           Dba:0x00000000
------------------------------------------------------------------------
 bda29137 01000000 88020000 e8490000 10004b8f 02000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 0d000d00 0d000100 
 00000000 00000000 00000000 02000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED>  m /x 9202 offset 500  
 File: /oracle/app/oracle/oradata/TEST2/test_01.dbf (0)
 Block: 1                Offsets:  500 to 1011           Dba:0x00000000
------------------------------------------------------------------------
 92020000 e8490000 10004b8f 02000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 0d000d00 0d000100 00000000 00000000 
 00000000 02000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 
 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 

 <32 bytes per line>

BBED> p kcvfh.kcvfhckp
struct kcvfhckp, 36 bytes                   @484     
   struct kcvcpscn, 8 bytes                 @484     
      ub4 kscnbas                           @484      0x00323a13
      ub2 kscnwrp                           @488      0x0000
   ub4 kcvcptim                             @492      0x3791a2bd
   ub2 kcvcpthr                             @496      0x0001
   union u, 12 bytes                        @500     
      struct kcvcprba, 12 bytes             @500     
         ub4 kcrbaseq                       @500      0x00000292
         ub4 kcrbabno                       @504      0x000049e8
         ub2 kcrbabof                       @508      0x0010
   ub1 kcvcpetb[0]                          @512      0x02
   ub1 kcvcpetb[1]                          @513      0x00
   ub1 kcvcpetb[2]                          @514      0x00
   ub1 kcvcpetb[3]                          @515      0x00
   ub1 kcvcpetb[4]                          @516      0x00
   ub1 kcvcpetb[5]                          @517      0x00
   ub1 kcvcpetb[6]                          @518      0x00
   ub1 kcvcpetb[7]                          @519      0x00

BBED> verify
DBVERIFY - Verification starting
FILE = /oracle/app/oracle/oradata/TEST2/test_01.dbf
BLOCK = 1

Block 1 is corrupt
Corrupt block relative dba: 0x00000001 (file 0, block 1)
Bad check value found during verification
Data in bad block:
 type: 11 format: 2 rdba: 0x00000001
 last change scn: 0x0000.00000000 seq: 0x1 flg: 0x04
 spare1: 0x0 spare2: 0x0 spare3: 0x0
 consistency value in tail: 0x00000b01
 check value in block header: 0x8331
 computed block checksum: 0xe40


DBVERIFY - Verification complete

Total Blocks Examined         : 1
Total Blocks Processed (Data) : 0
Total Blocks Failing   (Data) : 0
Total Blocks Processed (Index): 0
Total Blocks Failing   (Index): 0
Total Blocks Empty            : 0
Total Blocks Marked Corrupt   : 1
Total Blocks Influx           : 0
Message 531 not found;  product=RDBMS; facility=BBED


BBED> sum apply 
Check value for File 0, Block 1:
current = 0x8d71, required = 0x8d71
```

修改完成后,打开数据库:

```

Connected to:
Oracle Database 11g Enterprise Edition Release 11.2.0.4.0 - 64bit Production
With the Partitioning and Real Application Testing options

SQL> recover datafile  7  ;
Media recovery complete.
SQL> alter database open  ;

Database altered.
```

