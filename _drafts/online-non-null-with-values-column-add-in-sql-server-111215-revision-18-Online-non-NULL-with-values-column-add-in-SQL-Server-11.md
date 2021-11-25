---
id: 1234
title: Online non-NULL with values column add in SQL Server 11
date: 2011-07-13T18:47:59+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/13/1215-revision-18/
permalink: /2011/07/13/1215-revision-18/
---
Prior to SQL Server 11 when you add a new non-NULLable column with default values to an existing table a size-of data operation occurs: every row in the table is updated to add the default value of the new column. For small tables this is insignificant, but for large tables this can be so problematic as to completely prohibit the operation. But starting with SQL Server 11 the operation is, in most cases, instantaneous: only the table metadata is changed, no rows are being updated.

Lets look at a simple example, we&#8217;ll create a table with some rows and then add a non-NULL column with default values. First create and populate the table:

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

In my case this returned 0xD900000001000000, which means slot 0 on page 0xD9 (aka. 217) of file 1, and my test database has the DB_ID 6. Hence the parameters to DBCC PAGE

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
m_freeData = 7236                   m_reservedCnt = 0                   <span style="color:red">m_lsn = (30:71:25)</span>
m_xactReserved = 0                  m_xdesId = (0:0)                    m_ghostRecCnt = 0
m_tornBits = 2135435720             DB Frag ID = 1                      

Allocation Status

GAM (1:2) = ALLOCATED               SGAM (1:3) = ALLOCATED              
PFS (1:1) = 0x60 MIXED_EXT ALLOCATED   0_PCT_FULL                        DIFF (1:6) = CHANGED
ML (1:7) = NOT MIN_LOGGED           

Slot 0 <span style="color:red">Offset 0x60 Length 15</span>

Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 15

Memory Dump @0x000000000AEBA060

0000000000000000:   <span style="color:red">10000c00 01000000 34020000 020000†††††††††††††........4......</span>

Slot 0 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 1                              
Slot 0 Column 2 Offset 0x8 Length 4 Length (physical) 4
someValue = 564                     
</pre>

Note the last LSN that updated the page (30:71:25) and the size of the record in slot 1. Now lets add a non-NULL column with default values:

<pre>alter table test add otherValue int not null default 42 with values;
</pre>

We can select from the table and see that the table was changed and the rows have value 42 for the newly added column:

<pre>select top(2) * from test;

id          someValue   otherValue
----------- ----------- -----------
1           564         42
2           387         42
</pre>

Yet if we inspect again the page, we can see that is unchanged:

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
m_freeData = 7236                   m_reservedCnt = 0                   <span style="color:red">m_lsn = (30:71:25)</span>
m_xactReserved = 0                  m_xdesId = (0:0)                    m_ghostRecCnt = 0
m_tornBits = 2135435720             DB Frag ID = 1                      

Allocation Status

GAM (1:2) = ALLOCATED               SGAM (1:3) = ALLOCATED              
PFS (1:1) = 0x60 MIXED_EXT ALLOCATED   0_PCT_FULL                        DIFF (1:6) = CHANGED
ML (1:7) = NOT MIN_LOGGED           


Slot 0 <span style="color:red">Offset 0x60 Length 15</span>
Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 15

Memory Dump @0x000000000E83A060
0000000000000000:   <span style="color:red">10000c00 01000000 34020000 020000†††††††††††††........4......</span>

Slot 0 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 1                              

Slot 0 Column 2 Offset 0x8 Length 4 Length (physical) 4
someValue = 564                     

<span style="color:red">Slot 0 Column 3 Offset 0x0 Length 4 Length (physical) 0
otherValue = 42 </span>
</pre>

Note how the page header is unchanged, the last LSN is still (30:71:25),proof that the page was not modified, and the physical record is unchanged and has the same size as before. Yet DBCC shows a Column 3 and its value 42! If you pay attention you&#8217;ll notice that the Column 3 though has an Offset 0x0 and a physical length of 0. Column 3 is somehow materialized out of thin air, as it does not physically exists in the record on this page. The &#8216;magic&#8217; is that the table metadata has changed and it now contains a column with a &#8216;default&#8217; value:

<pre>select pc.* from sys.system_internals_partitions p
	join sys.system_internals_partition_columns pc on p.partition_id = pc.partition_id
	where p.object_id = object_id('test');
</pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/07/onlineschema.png" alt="" title="onlineschema" width="400" class="aligncenter size-full wp-image-1219" />](http://rusanu.com/wp-content/uploads/2011/07/onlineschema.png)

Notice that <a href="http://msdn.microsoft.com/en-us/library/ms189600.aspx" target="_blank">sys.system_internals_partition_columns</a> now has two new columns that are SQL Server 11 specific: <tt>has_default</tt> and <tt>default_value</tt>. The newly added column (the third one) has a default with value 42. This is how SQL Server 11 knows how to create the Column 3 for this record, even though is physically missing on the page. With this &#8216;magic&#8217; in place the ALTER TABLE will no longer have to update every row in the table and the operation is fast, metadata-only, no matter the number of rows in the table. This new behavior occurs automatically, no special syntax or setting is required, the engine will simply do the right thing. There is no penalty from having a missing value in a row. The &#8216;missing&#8217; value can be queried, updated, indexed, exactly as if the update during ALTER TABLE really occurred. There is no measurable performance penalty from having a default value.

What happens when we update a row? The &#8216;default&#8217; value is pushed into the row, even if the column was not modified. Consider this update:

<pre>update test set someValue = 565 where id = 1;
</pre>

Although we did not touch the otherValue column, the row now was modified and it contains the materialized value:

<pre>dbcc page(6,1,217,3)

...
m_freeData = 7240                   m_reservedCnt = 0                   <span style="color:red">m_lsn = (31:271:2)</span>
...
Slot 0 <span style="color:red">Offset 0x1c35 Length 19</span>

Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 19

Memory Dump @0x000000000AB8BC35

0000000000000000:   <span style="color:red">10001000 01000000 35020000 <b>2a000000</b> 030000††††........5...*......</span>

Slot 0 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 1                              

Slot 0 Column 2 Offset 0x8 Length 4 Length (physical) 4
someValue = 565                     

<span style="color:red">Slot 0 Column 3 Offset 0xc Length 4 Length (physical) 4
otherValue = 42</span>         

KeyHashValue = (8194443284a0)       
Slot 1 Offset 0x60 Length 15
Record Type = PRIMARY_RECORD        Record Attributes =  NULL_BITMAP    Record Size = 15

Memory Dump @0x000000000AB8A060
0000000000000000:   10000c00 02000000 83010000 020000†††††††††††††..............

Slot 1 Column 1 Offset 0x4 Length 4 Length (physical) 4
id = 2                              

Slot 1 Column 2 Offset 0x8 Length 4 Length (physical) 4
someValue = 387                     

<span style="color:blue">Slot 1 Column 3 Offset 0x0 Length 4 Length (physical) 0
otherValue = 42</span>         
</pre>

Notice how the physical record has increased in size (19 bytes vs. 15), the record has the value 42 in it (the hex 2a000000) and the Column 3 now has a real offset and physical size. So the update has trully materialized the default value in the row image. I intentionally left the next slot in the output this time to show that the record with id=2 was unaffected, it continues to have a smaller size of 15 bytes and Column 3 has no physical length.

## Default value vs. Default constraint

Is worth saying that the new SQL Server 11 default column value is not the same as the default value constraint. The default value is captured when the ALTER TABLE statement is run and can never change. Only rows existing in the table at the time of running ALTER TABLE statement will have missing &#8216;default&#8217; values. By contrast the default **constraint** can be dropped or modified and _new_ rows inserted after the ALTER TABLE will always have a value in present in row for the new column. Any REBUILD operation on the table (or on the clustered index) will materialize all the missing values as the rows are being copied from the old hobt to the new hobt. The new hobt columns (<tt>sys.system_internals_partition_columns</tt>) will loose the <tt>has_default</tt> and <tt>default_value</tt> attributes, in effect loosing any trace that this column was added online. A default constraint by contrast will be preserved as a table is rebuilt.

## Restrictions

Not all data types and default values can be added online. BLOB values like varchar(max), nvarchar(max), varbinary(max) and XML cannot be added online (and frankly I see no valid data model that has a non-NULL BLOB with a default&#8230;). Types that cannot be converted to sql\_variant cannot be added online, like hierarchy\_id, geometry and geography or user CLR based UDTs. Default expressions that require a different value for each row, like <tt>NEWID</tt> or <tt>NEWSEQUENTIALID</tt> cannot be added online (the default expression has to be a _runtime constant_, not to be confused with a deterministic expression, see <a href="http://blogs.msdn.com/b/conor_cunningham_msft/archive/2010/04/23/conor-vs-runtime-constant-functions.aspx" target="_blank">Conor vs. Runtime Constant Functions</a> for more details). In the case when the newly added column increases the maximum possible row size over the 8040 bytes limit the column cannot be added online. And is an Enterprise Edition only feature. For all the cases above the behavior will revert to adding the column &#8216;offline&#8217;, by updating every row in the table during the ALTER TABLE statement, creating a size-of-data update. When such a situation occurs a new XEvent is fired, which contains the reason why a size-of-data update occurred: <tt>alter_table_update_data</tt>.