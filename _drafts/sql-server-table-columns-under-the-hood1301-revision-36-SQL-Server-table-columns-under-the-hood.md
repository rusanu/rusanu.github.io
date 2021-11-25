---
id: 1358
title: SQL Server table columns under the hood
date: 2011-10-17T23:30:14+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/10/17/1301-revision-36/
permalink: /2011/10/17/1301-revision-36/
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
order by index_id, partition_number;
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
    
    ### Table structure changes
    
    Next lets look at some table structure modifying operations: adding, dropping and modifying columns. Were going to start anew with a fresh table for our examples:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
create table users  (
	user_id int not null identity(1,1) primary key,
	first_name char(100) null,
	last_name char(100) null,
	birth_date datetime null);
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed.png" alt="" title="physical-table-fixed" width="600" class="aligncenter size-full wp-image-1329" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed.png)
    
    Next lets modify some columns: we want to make the birth\_date not nullable and reduce the length of the first\_name to 75:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users alter column birth_date datetime not null;
alter table users alter column first_name char(75);
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-2.png" alt="" title="physical-table-fixed-2" width="600" class="aligncenter size-full wp-image-1330" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-2.png)
    
    Notice how the two alter operations ended up in adding a new column each, and dropping the old column. But also note how the null bit and the leaf_offset values have stayed _the same_. This means that the column was added &#8216;in place&#8217;, replacing the dropped column. This is a metadata only operation that did not modify any record in the table, it simply changed how the data in the existing records is interpreted.
    
    But now we figure out the 75 characters length is wrong and we want to change it back to 100. Also, the birth_date column probably doesn&#8217;t need hours, so we can change it to a date type:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users alter column birth_date date not null;
alter table users alter column first_name char(100);
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-3.png" alt="" title="physical-table-fixed-3" width="600" class="aligncenter size-full wp-image-1334" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-3.png)
    
    The birth\_date column has changed type and now it requires only 3 bytes, but the change occurred in-place, just as the nullability change we did before: it remains at the same offset in the record and it has the same null bit. However, the first\_name column was moved from offset 8 to offset 211, and the null bit was changed from 4 to 5. Because the first_name column has _increased_ in size it cannot occupy the same space as before in the record and the record has effectively increased to accommodate the _new_ first\_name column. This happened despite the fact that the first\_name column was originally of size 100 so in theory it could reclaim the old space it used in the record, but the is simply a too corner case for the storage engine to consider.
    
    By now we figure that the fixed length 100 for the first\_name and last\_name columns was a poor choice, so we would like to change them to more appropriate variable length columns:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users alter column first_name varchar(100);
alter table users alter column last_name varchar(100);
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-4.png" alt="" title="physical-table-fixed-4" width="600" class="aligncenter size-full wp-image-1337" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-4.png)
    
    The type change from fixed length column to variable length column cannot be done in-place, so the first\_name and last\_name columns get new null bit values.They also have negative leaf_offset values, which is typical for variable length columns as they don&#8217;t occupy fixed positions in the record.
    
    Next lets change the length of the variable columns: 
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users alter column first_name varchar(250);
alter table users alter column last_name varchar(250);
go
</code></pre> 
    
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-5.png" alt="" title="physical-table-fixed-5" width="600" class="aligncenter size-full wp-image-1339" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-5.png)
    
    For this change the column length was modified _without_ dropping the column and adding a new one. An increase of size for a variable length columns is one of the operations that is really _altering_ the physical column, not dropping the column and adding a new one to replace it. However, a decrease in size, or a nullability change, does again drop and add the column, as we can quickly check now:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users alter column first_name varchar(100);
alter table users alter column last_name varchar(250) not null;
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-6.png" alt="" title="physical-table-fixed-6" width="600" class="aligncenter size-full wp-image-1340" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-6.png)
    
    Finally, lets say we tried to add two more fixed length columns, but we we undecided on the name and length so we added a couple of columns, then deleted them and added again a new one:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users add mid_name char(50);
alter table users add surname char(25);
alter table users drop column mid_name;
alter table users drop column surname;

alter table users add middle_name char(100);
go&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-8.png" alt="" title="physical-table-fixed-8" width="600" class="aligncenter size-full wp-image-1346" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-8.png)
    
    This case is interesting because it shows how adding a new fixed column _can_ reuse the space left in the row by dropped fixed columns, but only if the dropped columns are the last fixed columns. In our table the columns <tt>mid_name</tt> and <tt>surname</tt> where originally added at offset 211 and 261 respectively and their length added up to 75 bytes. After we dropped them, the <tt>middle_name</tt> column we added is placed at offset 211, thus reusing the space formerly occupied by the dropped columns. This happens even though the length of the newly added column is 100 bytes, bigger than the 75 bytes occupied by the dropped columns before.
    
    By now, our 5 column table has actually 15 columns in the physical storage format, 10 dropped columns and 5 usable columns. The table uses 412 bytes of fixed space for the 3 fixed length columns that have a total length of only 112 bytes. The variable length columns that now store the first\_name and last\_name are stored in the record _after_ these 412 reserved bytes for fixed columns that are now dropped. Since the records always consume all the reserved fixed size, this is quite wasteful. How do we reclaim it? Rebuild the table:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table users rebuild;
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-7.png" alt="" title="physical-table-fixed-7" width="600" class="aligncenter size-full wp-image-1341" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-fixed-7.png)
    
    As you can see the rebuild operation got rid off the dropped columns and now the physical storage layout is compact, aligned with the logical layout. So whenever doing changes to a table structure remember that at the storage layer most changes are cumulative, they are most times implemented by dropping a column and adding a new column back, with the new type/length/nullability. Whenever possible the a modified column reuses the space in the record and newly added columns may reuse space previously used by dropped columns.
    
    ### Partitioned Tables
    
    When partitioning is taken into account another dimension of the physical table structure is revealed. To start, lets consider a typical partitioned table:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
create partition function pf (date) as range for values ('20111001');
go

create partition scheme ps as partition pf all to ([PRIMARY]);
go

create table sales (
	sale_date date not null,
	item_id int not null,
	store_id int not null,
	price money not null)
	on ps(sale_date);
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-partitioned-1.png" alt="" title="physical-table-partitioned-1" width="600" class="aligncenter size-full wp-image-1350" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-partitioned-1.png)
    
    As we know, the partitioned tables are, from a physical point of view, a collection of rowsets that belong to the same logical object (the 'table'). Looking under the hood shows that our table has two rowsets, and they have identical column structure. Typically new partitions are added by the ETL process that uploads data into a staging table that gets switched in using a fast partition switch operation:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
create table sales_staging (
	sale_date date not null,
	item_id int not null,
	store_id int not null,
	price numeric(10,2) not null,	
	constraint check_partition_range 
		check (sale_date = '20111002'))
	on [PRIMARY];


alter partition scheme ps next used [PRIMARY];
alter partition function pf() split range ('20111002');
go

alter table sales_staging switch to sales partition $PARTITION.PF('20111002');
go
&lt;span style="color:red">Msg 4944, Level 16, State 1, Line 2&lt;/span>
&lt;/pre>
&lt;p>&lt;span style="color:red">ALTER TABLE SWITCH statement failed because column 'price' has data type numeric(10,2) in source table 'test.dbo.sales_staging' which is different from its type money in target table 'test.dbo.sales'.&lt;/span>&lt;br />
</code>
    
    OK, so we made a mistake in the staging table, so lets correct it, then try to switch it in again:
    
    
<code class="prettyprint lang-sql">
&lt;pre>
alter table sales_staging alter column price money not null;
alter table sales_staging switch to sales partition $PARTITION.PF('20111002');
go
&lt;/pre>
&lt;p></code>  
    [<img src="http://rusanu.com/wp-content/uploads/2011/10/physical-table-partitioned-2.png" alt="" title="physical-table-partitioned-2" width="600" class="aligncenter size-full wp-image-1354" />](http://rusanu.com/wp-content/uploads/2011/10/physical-table-partitioned-2.png)
    
    We see that the partition switch operation has switched in the rowset of the staging table into the second partition of the <tt>sales</tt> table. But since the staging table had an operation that modified a column type, which in effect has dropped the numeric price columns and added a new price column of the appropriate money type, the second rowset of the partitioned table now has 5 columns, including a dropped one. The partition switch operation brings into the partitioned table the staging table _as is_, dropped columns and all. What that means is that partitions of the partitioned table can have, under the hood, a completely different physical structure from one another.