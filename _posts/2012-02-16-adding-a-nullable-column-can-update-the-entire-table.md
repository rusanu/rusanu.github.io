---
id: 1438
title: Adding a nullable column can update the entire table
date: 2012-02-16T15:31:25+00:00
author: remus
layout: post
guid: /?p=1438
permalink: /2012/02/16/adding-a-nullable-column-can-update-the-entire-table/
categories:
  - Samples
  - SQL 2012
  - Troubleshooting
---
In a previous article [Online non-NULL with values column add in SQL Server 2012](/2011/07/13/online-non-null-with-values-column-add-in-sql-server-11/) I talked about how adding a non-null column with default values is now an online operation in SQL Server 2012 and I mentioned how the situation when the newly added column may increase the rowsize can result in the operation being performed offline:

> In the case when the newly added column increases the maximum possible row size over the 8060 bytes limit the column cannot be added online.

In this article I want to show you how such a situation can arise and how it impacts even the case that prior to SQL Server 2012 was always online, namely adding a nullable column. Lets consider the following example:

<!--more-->

<pre><code class="prettyprint lang-sql">
create table T (
	c char(1) null,
	filler char(7998) null, 
	vc1 varchar(50) null,
	vc2 varchar(50) null);
go	

insert into  T (c, vc1, vc2) values 
	('A', NULL, NULL),
	('B', replicate ('X', 50), NULL),
	('C', NULL, replicate('Y', 50)),
	('D', replicate ('X', 50), replicate('Y', 50));
go

alter table T add some_int int null;
go
</code></pre>

If you run the snippet above in SQL Server 2005, SQL Server 2008 or SQL Server 2008 R2 it will succeed, and the column <tt>some_int</tt> will be added online, as a metadata only operation. However, the resulting table has some interesting properties. Lets try to update the newly added <tt>some_int</tt> column:

<pre><code class="prettyprint lang-sql">
update T set some_int = null
	where c= 'A';
update T set some_int = null
	where c= 'B';
update T set some_int = null
	where c= 'C';
update T set some_int = null
	where c= 'D';
go

(1 row(s) affected)

(1 row(s) affected)

(1 row(s) affected)
&lt;span style="color:red">Msg 511, Level 16, State 1, Line 7
Cannot create a row of size 8064 which is greater than the allowable maximum row size of 8060.&lt;/span>
The statement has been terminated.
</code></pre>

3 updates succeeded, the fourth failed. Well, I&#8217;ve been warned when I created the table, right? How about the message `Warning: The table "T" has been created, but its maximum row size exceeds the allowed maximum of 8060 bytes. INSERT or UPDATE to this table will fail if the resulting row exceeds the size limit`? But I&#8217;m updating a **fixed length** column and, furthermore, I&#8217;m updating it from NULL to NULL. Lets try something else:

<pre><code class="prettyprint lang-sql">
select * into T2 from T;	
go
&lt;span style="color:red">Msg 511, Level 16, State 1, Line 1
Cannot create a row of size 8064 which is greater than the allowable maximum row size of 8060.&lt;/span>
The statement has been terminated.
</code></pre>

OK, how about this:

<pre><code class="prettyprint lang-sql">
alter table T rebuild;
go
&lt;span style="color:red">Msg 511, Level 16, State 1, Line 1
Cannot create a row of size 8064 which is greater than the allowable maximum row size of 8060.&lt;/span>
The statement has been terminated.
</code></pre>

So my previous sequence of actions led to a table that I cannot rebuild! Clearly, not a very desirable state.

## SQL Server 2012 nullable ADD COLUMN

In SQL Server 2012 the situation above cannot happen. The adding of the column would be blocked: 

<pre><code class="prettyprint lang-sql">
create table T (
	c char(1) null,
	filler char(7998) null, 
	vc1 varchar(50) null,
	vc2 varchar(50) null);
go	

insert into  T (c, vc1, vc2) values 
	('A', NULL, NULL),
	('B', replicate ('X', 50), NULL),
	('C', NULL, replicate('Y', 50)),
	('D', replicate ('X', 50), replicate('Y', 50));
go

alter table T add some_int int null;
go

&lt;span style="color:red">Msg 511, Level 16, State 1, Line 1
Cannot create a row of size 8064 which is greater than the allowable maximum row size of 8060.&lt;/span>
The statement has been terminated.
</code></pre>

So SQL Server 2012 is able to detect the problem upfront and prevent the table from reaching the state in which it cannot be rebuilt. But detecting this situation is not a straightforward matter of table metadata and column size, the situation arises only if the _data_ in the table has this problem. After all the following sequence succeeds:

<pre><code class="prettyprint lang-sql">
create table T (
	c char(1) null,
	filler char(7998) null, 
	vc1 varchar(50) null,
	vc2 varchar(50) null);
go	

insert into  T (c, vc1, vc2) values 
	('A', NULL, NULL),
	('B', replicate ('X', 50), NULL),
	('C', NULL, replicate('Y', 50));
go

alter table T add some_int int null;
go
</code></pre>

Indeed it is only the record &#8216;D&#8217; that has a problem because both variable columns <tt>vc1</tt> and <tt>vc2</tt> have values in the row that are not NULL (and of a certain size). The ALTER TABLE &#8230; ADD COLUMN in this situation _must_ validate each row to ensure that it fits in the page, including the newly added column size. Therefore every row gets updated and the newly added column value of NULL is populated in the row image, thus enforcing that every row fits in the table. If any row grows over the maximum size due to the newly added column then the ALTER fails and all the rows updated thus far are rolled back.

> **If adding a nullable column in SQL Server 2012 has the potential of increasing the row size over the 8060 size then the ALTER performs an offline size-of-data update to every row of the table to ensure it fits in the page. This behavior is new in SQL Server 2012.**

This situation can arise when adding a new non-sparse fixed length column or a variable non-nullable column with default value to a table that already has the potential of creating rows over the maximum size of 8060. This new behavior (update every row in order to validate they all fit after the column is added) does not occur when adding a nullable variable length column or a sparse fixed length column.