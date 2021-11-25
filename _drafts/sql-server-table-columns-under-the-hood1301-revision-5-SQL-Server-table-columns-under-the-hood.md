---
id: 1308
title: SQL Server table columns under the hood
date: 2011-09-24T16:57:24+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/09/24/1301-revision-5/
permalink: /2011/09/24/1301-revision-5/
---
You probably can easily answer a question like &#8216;What columns does this table have?&#8217;. Whether you use the SSMS object explorer, or <tt>sp_help</tt>, or you query <tt>sys.column</tt>, the answer is fairly easy to find. But what is I ask &#8216;What are the **physical** columns of this table?&#8217;. Huh? Is there any difference? Lets see.

At the logical layer tables have exactly the structure you declare it in your CREATE TABLE statement, and perhaps modifications from ALTER TABLE statements. This is the layer at which you can look into <tt>sys.columns</tt> and see the table structure, or look at the table in SSMS object explorer and so on and so forth. But there is also a lower layer, the physical layer of the storage engine where the table might have surprisingly different structure from what you expect.

### Inspecting the physical table structure

To view the physical table structure you must use the undocumented system internals views: [<tt>sys.system_internals_partitions</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx) and [<tt>sys.system_internals_partition_columns</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx):


<code class="prettyprint lang-sql">
&lt;pre>
select p.index_id, p.partition_number, 
	pc.leaf_null_bit,
	coalesce(c.name, cx.name) as column_name,
	pc.partition_column_id,
	pc.max_inrow_length,
	pc.max_length,
	pc.key_ordinal,
	pc.leaf_offset,
	pc.is_nullable,
	pc.is_dropped,
	pc.is_uniqueifier,
	pc.is_sparse,
	pc.is_anti_matter
from sys.system_internals_partitions p
join sys.system_internals_partition_columns pc
	on p.partition_id = pc.partition_id
left join sys.index_columns ic
	on p.object_id = ic.object_id
	and ic.index_id = p.index_id
	and ic.index_column_id = pc.partition_column_id
left join sys.columns c
	on p.object_id = c.object_id
	and ic.column_id = c.column_id	
left join sys.columns cx
	on p.object_id = cx.object_id	
	and p.index_id in (0,1)
	and pc.partition_column_id = cx.column_id
where p.object_id = object_id('...')
order by index_id, partition_number, leaf_null_bit;
&lt;/pre>
&lt;p></code>

Lets inspect some simple table structures:


<code class="prettyprint lang-sql">
&lt;pre>
create table users (
	user_id int not null identity(1,1),
	first_name varchar(100) null,
	last_name varchar(100) null,
	birth_date datetime null);
&lt;/pre>
&lt;p></code>

Running our query after, of course, we specify <tt>object_id('users')</tt>:

[<img src="http://rusanu.com/wp-content/uploads/2011/09/physical-table-1.png" alt="" title="physical-table-1" width="600" class="aligncenter size-full wp-image-1304" />](http://rusanu.com/wp-content/uploads/2011/09/physical-table-1.png)