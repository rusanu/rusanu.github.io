---
id: 1323
title: SQL Server table columns under the hood
date: 2011-09-25T16:26:07+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/09/25/1301-revision-16/
permalink: /2011/09/25/1301-revision-16/
---
You probably can easily answer a question like &#8216;What columns does this table have?&#8217;. Whether you use the SSMS object explorer, or <tt>sp_help</tt>, or you query <tt>sys.column</tt>, the answer is fairly easy to find. But what is I ask &#8216;What are the **physical** columns of this table?&#8217;. Huh? Is there any difference? Lets see.

At the logical layer tables have exactly the structure you declare it in your CREATE TABLE statement, and perhaps modifications from ALTER TABLE statements. This is the layer at which you can look into <tt>sys.columns</tt> and see the table structure, or look at the table in SSMS object explorer and so on and so forth. But there is also a lower layer, the physical layer of the storage engine where the table might have surprisingly different structure from what you expect.

### Inspecting the physical table structure

To view the physical table structure you must use the undocumented system internals views: [<tt>sys.system_internals_partitions</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx) and [<tt>sys.system_internals_partition_columns</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx):


<code class="prettyprint lang-sql">
&lt;pre>
select p.index_id, p.partition_number, 
	pc.leaf_null_bit,
	coalesce(cx.name, c.name) as column_name,
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

We can see that the physical structure is very much as we expected: the physical rowset has 4 physical columns, of the expected types and sizes. One thing to notice is that, although the column order is the one we specified, the columns are layout on disk in a different order: the <tt>user_id</tt> is stored in row at offset 4, followed by the <tt>birth_date</tt> at offset 8 and then by the two variable length columns (<tt>first_name</tt> and <tt>last_name</tt>). This should be of no surprise as we know that the row format places all fixed length columns first in the row, ahead of the variable length columns.

### Adding a clustered index

Our table right now is a heap, we should make <tt>user_id</tt> a primary key, and lets check the table structure afterward:


<code class="prettyprint lang-sql">
&lt;pre>
alter table users add constraint pk_users_user_id primary key (user_id);
&lt;/pre>
&lt;p></code>  
[<img src="http://rusanu.com/wp-content/uploads/2011/09/physical-table-pk.png" alt="" title="physical-table-pk" width="600" class="aligncenter size-full wp-image-1309" />](http://rusanu.com/wp-content/uploads/2011/09/physical-table-pk.png)

As we can see, the table has changed from a heap into a clustered index (index_id changed from 0 to 1) and the <tt>user_id</tt> column has become part of the key.

### Non-Unique clustered indexes

Now lets say that we want a different clustered index, perhaps by <tt>birth_date</tt> (maybe our application requires frequent range scans on this field). We would change the primary key into a non-clustered primary key (remember, the primary key and the clustered index do **not** have to be the same key) and add a clustered index by the <tt>birth_date</tt> column:


<code class="prettyprint lang-sql">
&lt;pre>
alter table users drop constraint pk_users_user_id;
create clustered index cdx_users on users (birth_date);	
alter table users add constraint 
	pk_users_user_id primary key nonclustered  (user_id);
&lt;/pre>
&lt;p></code>  
[<img src="http://rusanu.com/wp-content/uploads/2011/09/physical-table-cdx.png" alt="" title="physical-table-cdx" width="600" class="aligncenter size-full wp-image-1310" />](http://rusanu.com/wp-content/uploads/2011/09/physical-table-cdx.png)

Several changes right there:

  * A new index has appeared (index_id 2), which was expected, this is the non-clustered index that now enforces the primary key constraint on column <tt>user_id</tt>.
  * The table has a new physical column, with <tt>partition_column_id</tt> 0. This new column has no name, because it is not visible in the logical table representation (you cannot select it, nor update it). This is an [uniqueifier column](http://msdn.microsoft.com/en-us/library/ms190639.aspx) (<tt>is_uniqueifier</tt> is 1) because we did not specify that our clustered index in unique:  
    > If the clustered index is not created with the UNIQUE property, the Database Engine automatically adds a 4-byte uniqueifier column to the table. When it is required, the Database Engine automatically adds a uniqueifier value to a row to make each key unique. This column and its values are used internally and cannot be seen or accessed by users.
    
    Klaus Aschenbrenner has [a good series of articles that explain uniqueifier columns in more detail](http://www.csharp.at/blog/PermaLink,guid,98d6366f-9e37-4413-8435-793129ac87cb.aspx).</li> 
    
      * The non-clustered index that enforces the primary key constraint has **3** physical columns: <tt>user_id</tt> and two unnamed ones. This is because each row in the non-clustered index contains the corresponding clustered index row key values, in our case the <tt>birth_date</tt> column and the uniqueifier column.</ul> 
    
    ### Online operations and anti-matter columns
    
    Is the uniqueifier column the only hidden column in a table? No. Lets rebuild our non-clustered index, making sure we specify it to be an online operation:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter index pk_users_user_id on users rebuild with (online=on);
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/09/physical-table-anti-matter.png" alt="" title="physical-table-anti-matter" width="600" class="aligncenter size-full wp-image-1317" />](http://rusanu.com/wp-content/uploads/2011/09/physical-table-anti-matter.png)
    
    Notice how the non-clustered index now has one more column, a new column that has the <tt>is_antimatter</tt> value 1. As you probably guess, this is an anti-matter column. For an in-depth explanation of what is the purpose of the anti-matter column I recommend reading [Online Indexing Operations in SQL Server 2005](http://msdn.microsoft.com/en-us/library/cc966402.aspx):
    
    > During the build phase, rows in the new &#8220;in-build&#8221; index may be in an intermediate state called antimatter. This mechanism allows concurrent DELETE statements to leave a trace for the index builder transaction to avoid inserting deleted rows. At the end of the index build operation all antimatter rows should be cleared.
    
    Note that even after the online operation finishes and the antimatter rows are removed, the antimatter _column_ will be part of the physical rowset structure.
    
    There could exist more hidden columns in the table, for example if we enable change tracking:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users ENABLE CHANGE_TRACKING;
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/09/physical-table-xdes.png" alt="" title="physical-table-xdes" width="600" class="aligncenter size-full wp-image-1322" />](http://rusanu.com/wp-content/uploads/2011/09/physical-table-xdes.png)
    
    Enabling change tracking on our table has added one more hidden column.
    
    As you can see, the physical shape of the table can differ significantly from the logical table layout. The examples above showed how hidden columns are added by the storage engine to the table to track uniqueness, support online operations or to enable change tracking.