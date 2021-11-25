---
id: 2042
title: How to update a table with a columnstore index
date: 2013-08-02T03:43:17+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/02/1202-autosave/
permalink: /2013/08/02/1202-autosave/
---
In my previous article [How to use columnstore indexes in SQL Server](http://rusanu.com/2011/07/13/how-to-use-columnstore-indexes-in-sql-server/) we&#8217;ve seen how to create a columnstore index on a table and how certain queries can significantly reduce the IO needed and thus increase in performance by leveraging this new feature. But once a columnstore index is added to a table the table becomes read-only as it cannot be updated. Trying to insert a new row in the table will result in an error:

<pre><code class="prettyprint lang-sql">
insert into sales ([date],itemid, price, quantity) values ('20110713', 1,1.0,1);
</code></pre>

<span style="color:red">Msg 35330, Level 15, State 1, Line 1<br /> INSERT statement failed because data cannot be updated in a table with a columnstore index. Consider disabling the columnstore index before issuing the INSERT statement, then rebuilding the columnstore index after INSERT is complete.<br /> </span>

<!--more-->

The error message recommends a &#8216;workaround&#8217;, but rebuilding the columnstore index for updates may be prohibitively expensive. For the DW and BI scenarios that columnstore indexes are targeting there is a much better solution: use table partitioning. With SQL Server 11 the limit of maximum 1000 partitions per table has been increased to 15000 partitions and with this new limit one can configure the ETL process to update every day into a new partition and still retain many many years of data. The ETL process can upload the daily data into a staging table, create a columnstore index on the staging table, then use the fast ALTER TABLE &#8230; SWITCH operation to &#8216;switch in&#8217; the new data. Using the very same example as in my previous article, lets create a staging table with identical structure as the sales facts table:

<pre><code class="prettyprint lang-sql">
create table sales_staging (
	[id] int not null identity (1000000,1),
	[date] date not null,
	itemid smallint not null,
	price money not null,
	quantity numeric(18,4) not null,
	constraint check_date check ([date] = '20110716')) on [PRIMARY];
go

create unique clustered index cdx_sales_staging_date_id 
   on sales_staging ([date], [id]) on [PRIMARY];
go
</code></pre>

Note how the staging table has a constraint check that enforces the date to be the valid date for the next partition to be switched in. Now lets populate the staging table with some more dummy sales facts:

<pre><code class="prettyprint lang-sql">
set nocount on
go

declare @i int = 0;
begin transaction;
while @i &lt; 250000
begin
	insert into sales_staging ([date], itemid, price, quantity) 
		values ('20110716', rand()*10000, rand()*100 + 100, rand()* 10.000+1);
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
</code></pre>

Now that our fake ETL process has finished preparing the last days sales data into a staging table, lets add a columnstore index identical with the one on the real sales table:

<pre><code class="prettyprint lang-sql">
create columnstore index cs_sales_price_staging 
          on sales_staging ([date], itemid, price, quantity);
go
</code></pre>

OK, our staging table is complete so lets switch it in into the 'big' sales table:

<pre><code class="prettyprint lang-sql">
alter partition scheme ps next used [PRIMARY];
alter partition function pf() split range ('20110717');
go

alter table sales_staging switch to sales partition $PARTITION.PF('20110716');
go
</code></pre>

That's it! We've just updated our sales table with the sales fact for the last day, despite the fact that it contained a columnstore index, **without disabling the columnstore index**. The increased partitions count supported in SQL Server 11 combined with the fact that aligned columnstore indexes are supported for the fast partition switch operations makes the tables with column store indexes updateable in practice, if the ETL process uses a staging table and the ETL schedule matches the partitioning scheme.