---
id: 1198
title: How to use columnstore indexes in SQL Server
date: 2011-07-13T00:03:10+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/13/1189-revision-6/
permalink: /2011/07/13/1189-revision-6/
---
Column oriented storage is the data storage of choice for business analysis application. Column oriented storage allows for a high data compression rate and as such it can increase processing speed by reducing the IO needs. Now SQL Server allows for creating column oriented indexes (called COLUMNSTORE indexes) and thus brings the benefits of this highly efficient BI oriented indexes in the same engine that runs the OLTP workload. The syntax for creating columnstore indexes is described on MSDN at <a href="http://msdn.microsoft.com/en-us/library/gg492153%28v=SQL.110%29.aspx" target="_blank">CREATE COLUMNSTORE INDEX</a>. Lets walk trough a very simple example of how to create and use a columnstore index. First lets have a dummy sales table:

<pre>create partition function pf (date) as range left for values 
  ('20110712', '20110713', '20110714', '20110715', '20110716');
go

create partition scheme ps as  partition pf all to ([PRIMARY]);
go

create table sales (
	[id] int not null identity (1,1),
	[date] date not null,
	itemid smallint not null,
	price money not null,
	quantity numeric(18,4) not null)
	on ps([date]);
go

create unique clustered index cdx_sales_date_id on sales ([date], [id]) on ps([date]);
go
</pre>

Notice how I created this table on a partitioning scheme that has one partition a day. I&#8217;ll explain shortly why I choose this particular arrangement. for now, lets populate the table with 1 million &#8216;sales&#8217; facts:

<pre>set nocount on;
go

declare @i int = 0;
begin transaction;
while @i &lt; 1000000
begin
	declare @date date = dateadd(day, @i /250000.00, '20110712');
	insert into sales ([date], itemid, price, quantity) 
		values (@date, rand()*10000, rand()*100 + 100, rand()* 10.000+1);
	set @i += 1;
	if @i % 10000 = 0
	begin
		raiserror (N'Inserted %d', 0, 1, @i);
		commit;
		begin tran;
	end
end
commit;
go
</pre>

If we look now at the structure of the sales table we see that each partition has 250k rows spread along 1089 pages:

<pre>select * from sys.system_internals_partitions p
	where p.object_id = object_id('sales');

select au.* from sys.system_internals_allocation_units au
	join sys.system_internals_partitions p 
		on p.partition_id = au.container_id
	where p.object_id = object_id('sales');
go
</pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/07/columnstore_cdx_size.png" alt="" title="columnstore_cdx_size" width="689" height="303" class="aligncenter size-full wp-image-1192" />](http://rusanu.com/wp-content/uploads/2011/07/columnstore_cdx_size.png)

If we now run a BI type of query like get the number of sales facts and the total sales for a day, the query would have to scan an entire partition, generating 1089 logical reads:

<pre>set statistics io on;
select count(*), sum(price*quantity) from sales where date = '20110713'
set statistics io off;
go

Table 'sales'. Scan count 1, logical reads 1089, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.
</pre>

So lets create a columnstore index on this table:

<pre>create columnstore index cs_sales_price on sales ([date], price, quantity) on ps([date]);
go
</pre>

If we look at the structure of the columnstore index we'll see that it has a much smaller footprint, only 362 pages:

<pre>select * from sys.system_internals_partitions p
	where p.object_id = object_id('sales')
	and index_id = 2;

select au.* from sys.system_internals_allocation_units au
	join sys.system_internals_partitions p 
		on p.partition_id = au.container_id
	where p.object_id = object_id('sales')
		and index_id = 2;
go
</pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/07/columnstore_size.png" alt="" title="columnstore_size" width="691" height="404" class="aligncenter size-full wp-image-1195" />](http://rusanu.com/wp-content/uploads/2011/07/columnstore_size.png)

Note how the columnstore index has no pages allocated for the IN\_ROW\_DATA allocation unit, but instead has pages allocated to the LOB_DATA allocation unit. So a columnstore index has no rows, instead it uses the BLOB storage to store the column 'segments'. Due to compression possible with column oriented storage, it needs only about one third of the pages needed by the clustered index, although it contains the same columns and the same number of sales facts. If we run again the very same query as before, we'll see how it uses the columnstore index and generates less IO:

<pre>set statistics io on;
select count(*), sum(price*quantity) from sales where date = '20110713'
set statistics io off;
go

Table 'sales'. Scan count 1, logical reads 358, physical reads 0, read-ahead reads 0, lob logical reads 0, lob physical reads 0, lob read-ahead reads 0.
</pre></p>