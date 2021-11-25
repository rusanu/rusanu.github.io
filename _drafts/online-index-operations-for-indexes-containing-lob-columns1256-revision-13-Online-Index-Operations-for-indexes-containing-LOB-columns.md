---
id: 1272
title: Online Index Operations for indexes containing LOB columns
date: 2011-08-05T13:38:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/05/1256-revision-13/
permalink: /2011/08/05/1256-revision-13/
---
SQL Server supports online index and table rebuild operations which allow for maintenance operations to occur w/o significant downtime. While a table is being rebuild the table is fully utilizable, it can be queried and updated. any updates done to the table while the online rebuild operation is occurring will be contained in the final rebuilt table. A detailed explanation on how these online rebuild operations work can be found in the <a href="http://technet.microsoft.com/en-us/library/cc966402.aspx" target="_blank">Online Indexing Operations in SQL Server 2005</a> white paper. But Online Index Build operations in SQL Server 2005, 2008 and 2008 R2 do not support tables that contain LOB columns, attempting to do so would trigger an error:

> <span style="color:red">Msg 2725, Level 16, State 2, Line &#8230;<br /> An online operation cannot be performed for index &#8216;&#8230;&#8217; because the index contains column &#8216;&#8230;&#8217; of data type text, ntext, image, varchar(max), nvarchar(max), varbinary(max), xml, or large CLR type. For a non-clustered index, the column could be an include column of the index. For a clustered index, the column could be any column of the table. If DROP_EXISTING is used, the column could be part of a new or old index. The operation must be performed offline.</span> 

To be accurate the restriction applies not to tables, but to any index or heap that contains an LOB column. That, of course, includes any clustered index or the base heap of a table if the table contains any LOB columns, but it would include any non-clustered index that includes a LOB column. In other words I can rebuild online non-clustered indexes of any table as long as they don&#8217;t use the INCLUDE clause to add a LOB column from the base table, but for sure I cannot rebuild online the table itself (meaning the clustered index or the heap).

Starting with SQL Server 11 it is actually permitted to rebuild online indexes and heaps containing LOB columns. The old legacy types (<tt>text</tt>, <tt>ntext</tt> and <tt>image</tt>) are not supported, not surprising considering that these types are on the deprecation path.

To understand why the original online rebuild operations from previous versions did not support LOB columns we need to consider the SQL Server <a href="http://msdn.microsoft.com/en-us/library/ms189051.aspx" target="_blank">Table and Index Organization</a>. All indexes and tables consist of three allocation units: one for row data, one for overflow row data and one for LOB data. We can see this if we inspect the <tt>sys.system_internals_allocation_units</tt> system catalog view:

<pre><code class="prettyprint lang-sql">
create table test (id int not null identity(1,1), 
	somevar1 varchar(6000),
	somevar2 varchar(6000),
	someblob varchar(max))
go

insert into test (somevar1, somevar2, someblob) values ('A', 'B', 'C');
insert into test (somevar1, somevar2, someblob) values 
         (replicate('A', 6000), replicate('B', 6000), replicate('C', 8000))
go

select au.*
from sys.system_internals_allocation_units au
join sys.system_internals_partitions p on au.container_id = p.partition_id
where p.object_id = object_id('test');
go
</code>
</pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/08/oiblob-au.png" alt="" title="oiblob-au" width="600" height="77" class="aligncenter size-full wp-image-1263" />](http://rusanu.com/wp-content/uploads/2011/08/oiblob-au.png)

Our test table shows three allocation units. Now lets rebuild our table and look again at our allocataon units:

<pre><code class="prettyprint lang-sql">
alter table test rebuild;
go

select au.*
from sys.system_internals_allocation_units au
join sys.system_internals_partitions p on au.container_id = p.partition_id
where p.object_id = object_id('test');
go
</code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/08/oiblob-au-offline.png" alt="" title="oiblob-au-offline" width="600" height="75" class="aligncenter size-full wp-image-1266" />](http://rusanu.com/wp-content/uploads/2011/08/oiblob-au-offline.png)

We can see that our DATa and SLOB (aka. row overflow) allocation units have changed (they have different IDs and start at different pages). But the important thing is that the BLOB allocation unit has **not** changed. after the offline table rebuild, it has the same ID and starts at the same pages. This is because table and index rebuild operations do not rebuild the LOB data. They rebuild the row data and the row-overflow data, but the newly built rows will simply point back to the same old LOB data. The idea is that tables with LOB columns have a_large_ LOB values and rebuilding the LOB data would be prohibitive, with little or no benefit.

Offline operations can avoid rebuilding the LOB data without problems, but for online index and table rebuilds this poses an issue: for the duration of the online rebuild operation both the old rowset (the old index/table) and the new rowset would point to the same LOB data _while updates are being made to rows_. The issue is that LOB data can be pulled in-row (LOB column data is moved from the LOB allocation unit to the ROW allocation unit) or pushed off-row (data is moved from ROW allocation unit into the LOB allocation unit) and the decision when and what to pull in row differs between the old rowset and the new rowset. At the end of the online rebuild operation the data in the LOB allocation unit may be referenced by the old rowset only, by the new rowset only, or by both rowsets. As the online operation completes and drops the old rowset, some of the data in the LOB allocation unit may be suddenly orphaned because it was only referenced by the old rowset. And since the online operation can also fail and rollback LOB data referenced only by the new rowset would become orphaned when the new rowset is dropped during rollback.

In SQL Server 11 this problem was solved and now online operations can rebuild indexes and tables with LOB columns without the risk of leaving orphaned data in the LOB allocation unit. SQL Server will internally track what LOB data would be orphaned if the old rowset or the new rowset is dropped and will take appropriate actions to avoid leaving orphaned data in the LOB allocation unit.

## Limitations

Partial LOB <tt>.WRITE</tt> updates are transformed into full updates.
:   LOB data supports a highly efficient update syntax, the <tt><a href="http://msdn.microsoft.com/en-us/library/ms177523.aspx" target="_blank>.WRITE</a></tt> syntax. This is critical in creating streaming semantics, see [Download and Upload images from SQL Server via ASP.Net MVC](http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/). When the <tt>.WRITE</tt> syntax is used on a LOB column belonging to an index that is being rebuilt online the generated plan will silently change it into a full value update, which generates significantly more log. If you rely heavily on this functionality be aware and schedule your online rebuilds accordingly.

DBCC CHECK operations will skip the consistency check of LOB allocation units belonging to indexes that are in the process of being rebuilt online.
:   During the online operation the LOB allocation unit is shared between the old index and the new index and is consistent if you consider _both_ owners, however it may look inconsistent if considered from either one of the owner point of view.

File SHRINK operation will skip pages belonging to LOB allocation units belonging to indexes that are in the process of being rebuilt online.
:   If LOB data is shrunk the pointers in the ROW data referencing the LOB data that had moved have to be updated. While an online index rebuild occurs there could be two sets of pointers referencing the same LOB data, one in the old rowset and one in the new rowset.