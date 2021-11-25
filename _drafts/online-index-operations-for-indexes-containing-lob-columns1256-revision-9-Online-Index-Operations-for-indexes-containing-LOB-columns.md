---
id: 1268
title: Online Index Operations for indexes containing LOB columns
date: 2011-08-05T12:04:00+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/05/1256-revision-9/
permalink: /2011/08/05/1256-revision-9/
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

We can see that our DATa and SLOB (aka. row overflow) allocation units have changed (they have different IDs and start at different pages). But the important thing is that the BLOB allocation unit has **not** changed.