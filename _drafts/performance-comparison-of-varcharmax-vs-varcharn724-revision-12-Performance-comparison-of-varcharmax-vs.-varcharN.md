---
id: 2056
title: Performance comparison of varchar(max) vs. varchar(N)
date: 2013-08-02T03:56:57+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/02/724-revision-12/
permalink: /2013/08/02/724-revision-12/
---
The question of comparing the MAX types (VARCHAR, NVARCHAR, VARBINARY) with their non-max counterparts is often asked, but the answer usually gravitate around the storage differences. But I&#8217;d like to address the point that these types have inherent, intrinsic performance differences that are not driven by different storage characteristics. In other words, simply comparing and manipulating variables and columns in T-SQL can yield different performance when VARCHAR(MAX) is used vs. VARCHAR(N).

### Assignment

First comparing simple assignment, assign a value to a VARBINARY(8000) variable in a tight loop:

<!--more-->

<pre><code class="prettyprint lang-sql">
declare @x varchar(8000);
declare @startTime datetime;
declare @i int;

set @i = 0;
set @startTime = getutcdate();

while @i &lt; 1000000
begin
  set @x = 'abc';
  set @i = @i + 1;
end

select datediff(ms, @startTime, getutcdate());
</code>
</pre>

This script runs on my test server in 4.9 seconds on average. Simply changing the variable declaration to VARBINARY(MAX) changes the run time to an average of of 9.2 seconds.

### Comparison

Next I measured the performance of a simple comparison:

<pre><code class="prettyprint lang-sql">
declare @x varchar(8000);
declare @startTime datetime;
declare @i int;

set @i = 0;
set @startTime = getutcdate();
set @x = 'abc';


while @i &lt; 1000000
begin
  declare @y bit;
  set @y = case when @x = 'abc' then 1 else 0 end;
  set @i = @i + 1;
end

select datediff(ms, @startTime, getutcdate());
</code></pre>

Average run time for VARCHAR(8000): 5.8 seconds. For VARCHAR(MAX): 6.5 seconds.

### Data Access

The next text does a simple string comparison of a VARCHAR(MAX) field vs. a VARCHAR(8000) field. The actual value stored is identical in both cases, a simple 3 letter string 'abc'. The data access does not retrieve the string value, it simply compares its value against a WHERE clause predicate:

<pre><code class="prettyprint lang-sql">
create table test (
  id int identity(1,1) primary key, 
  x varchar(max));
go

insert into test(x) values ('abc');
go

declare @x varchar(max);
declare @startTime datetime;
declare @i int;

set nocount on;
set @i = 0;
set @startTime = getutcdate();

while @i &lt; 100000
begin
  declare @id int;
  select @id = id 
    from test 
    where id = 1
    and x = 'abc';
  set @i = @i + 1;
end

select datediff(ms, @startTime, getutcdate());
</code></pre>

Loop execution time for VARCHAR(8000): 3.1 seconds. For VARCHAR(MAX): 3.6 seconds.

Furthermore, if we change the WHERE predicate to compare against a max type variable instead of the literal '<span style="color:red">abc</span>' the loop time increases to 3.9 seconds.

### Conclusion

The code path that handles the MAX types (varchar, nvarchar and varbinary) is different from the code path that handles their equivalent non-max length types. The non-max types can internally be represented as an ordinary pointer-and-length structure. But the max types cannot be stored internally as a contiguous memory area, since they can possibly grow up to 2Gb. So they have to be represented by a streaming interface, similar to COM's <a href="http://msdn.microsoft.com/en-us/library/aa380034%28VS.85%29.aspx" ref="nofollow" target="_blank">IStream</a>. This carries over to every operation that involves the max types, including simple assignment and comparison, since these operations are more complicated over a streaming interface. The biggest impact is visible in the code that allocates and assign max-type variables (my first test), but the impact is visible on every operation.

The impact is reasonable on most operations with the max type operations being about 10% slower. So is this something to worry about, or even worth blogging about, after all? Actually, in a recent project of mine I had to change a column type from varchar(max) to varchar(5000) to alleviate the impact of max-types performance. The code happened to be on a very very performance critical path and testing clearly showed the impact was not only measurable, was quite actually quite serious. The processing throughput increased by about 25% just from this minor change in my case.

Although much discussion exists around the storage of max types vs. the storage of non-max types and the similarities and differences between them (eg. in-row vs. out-of-row vs. row-overflow storage), there is also a more fundamental aspect that differentiate these types: SQL Server internal code differences. These differences manifest themselves even when the storage of the max type is optimized in-row, for small and very small values of the BLOB field.