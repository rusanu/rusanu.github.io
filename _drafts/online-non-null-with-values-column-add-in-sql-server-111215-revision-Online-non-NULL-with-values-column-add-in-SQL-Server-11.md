---
id: 1216
title: Online non-NULL with values column add in SQL Server 11
date: 2011-07-13T14:20:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/13/1215-revision/
permalink: /2011/07/13/1215-revision/
---
Prior to SQL Server 11 when you add a new non-NULLable column with default values to an existing table a size-of data operation occurs: every row in the table is updated to add the default value of the new column. For small tables this is insignificant, but for large tables this can be so problematic as to completely prohibit the operation. With SQL Server 11 the operation is, in most cases, instantaneous: only the table metadata is changed, no rows are being updated. Lets look at a simple example, we&#8217;ll create a table with some rows and then add a non-NULL column with default values:

<pre>create table test (
	id int not null identity(1,1) primary key,
	someValue int not null);
go

set nocount on;
insert into test (someValue) values (rand()*1000);
go 1000
</pre>

We can inspect the physical structure of the table&#8217;s records using DBCC PAGE. first lets find the page that contains the first record of the table:

<pre>select %%physloc%%, * from test where id = 1;
</pre>

In my case this returned 0xD900000001000000, which means slot 1 on page 0xD9 (aka. 217), and my test database has the DB_ID 6:

<pre>dbcc traceon (3604,-1)
dbcc page(6,1,217,3)

Page @0x0000000170D5E000

m_pageId = (1:217)                  m_headerVersion = 1                 m_type = 1
m_typeFlagBits = 0x0                m_level = 0                         m_flagBits = 0x200
m_objId (AllocUnitId.idObj) = 84    m_indexId (AllocUnitId.idInd) = 256 
Metadata: AllocUnitId = 72057594043432960                                
Metadata: PartitionId = 72057594039042048                                Metadata: IndexId = 1
Metadata: ObjectId = 245575913      m_prevPage = (0:0)                  m_nextPage = (1:220)
pminlen = 12                        m_slotCnt = 476                     m_freeCnt = 4
m_freeData = 7236                   m_reservedCnt = 0                   m_lsn = (30:71:25)
m_xactReserved = 0                  m_xdesId = (0:0)                    m_ghostRecCnt = 0
m_tornBits = 2135435720             DB Frag ID = 1                      

Allocation Status

GAM (1:2) = ALLOCATED               SGAM (1:3) = ALLOCATED              
PFS (1:1) = 0x60 MIXED_EXT ALLOCATED   0_PCT_FULL                        DIFF (1:6) = CHANGED
ML (1:7) = NOT MIN_LOGGED           

Slot 0 Offset 0x60 Length 15

Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 15

Memory Dump @0x000000000AEBA060

0000000000000000:   <span style="color:red">10000c00 01000000 34020000 020000†††††††††††††........4......</s

Slot 0 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 1                              
Slot 0 Column 2 Offset 0x8 Length 4 Length (physical) 4
someValue = 564                     

</pre>