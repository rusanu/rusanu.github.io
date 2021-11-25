---
id: 1294
title: T-SQL functions do no imply a certain order of execution
date: 2011-08-10T01:18:18+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/10/1290-revision-4/
permalink: /2011/08/10/1290-revision-4/
---
Looking at this question on StackOverflow: <a href="http://stackoverflow.com/questions/6989522/conversion-failed-when-converting-from-a-character-string-to-uniqueidentifier-err" target="_blank">Conversion failed when converting from a character string to uniqueidentifier error in SQL Server</a> one can see a reproducible example where a string split UDF works fine in the SELECT statement, but it gives a conversion error in the DELETE statement. Certainly, a bug, right? Actually, not.

The issue at hand is in fact remarkably similar to another common misconception around T-SQL I had to debunk some time ago, see [On SQL Server boolean operator short-circuit](http://rusanu.com/2009/09/13/on-sql-server-boolean-operator-short-circuit/): that C like operator short-circuit is guaranteed in T-SQL (hint: it isn&#8217;t, read the linked article to see a clear counter example).

The common misconception around T-SQL inline-table functions is that the function is evaluated somehow separately from the rest of the query and some sort of temporary result is created that is then used in the overall query execution. This understanding comes naturally to the imperative procedural language mindset of developers trained in C, C++, C# and other similar languages. But in the SQL Server declarative language that is T-SQL, your intuition is actually wrong. To illustrate I will give a simple counter-example, reusing the code from my earlier boolean short-circuit article:

<pre><code class="prettyprint lang-sql">
create table eav (
    eav_id int identity(1,1) primary key,
    attribute varchar(50) not null,
    is_numeric bit not null,
    [value] sql_variant null);
create index eav_attribute on eav(attribute) include ([value]);
go

-- Fill the table with random values
set nocount on
declare @i int;
select @i = 0;
begin tran
while @i &lt; 100000
begin
    declare @attribute varchar(50),
        @is_numeric bit,
        @value sql_variant;
    select @attribute = 'A' + cast(cast(rand()*1000 as  int) as varchar(3));
    select @is_numeric = case when rand() > 0.5 then 1 else 0 end;
    if 1=@is_numeric
        select @value = cast(rand() * 100 as int);
    else
        select @value = 'Lorem ipsum';
    insert into eav (attribute, is_numeric, [value])
    values (@attribute, @is_numeric, @value);
    select @i = @i+1;
    if (@i % 1000 = 0)
    begin
		commit;
		raiserror(N'Inserted %d', 0,0,@i);
		begin tran;
    end
end
commit
go

-- insert a 'trap'
insert into eav (attribute, is_numeric, [value])
values ('B1', 0, 'Gotch ya');
go
</code>
</pre>

Now we&#8217;re going to define our inline table UDF:

<pre><code class="prettyprint lang-sql">
create function udf_get_only_numerics()
returns table
with schemabinding
as return 
	select [value], [attribute]
	from dbo.eav
	where is_numeric = 1;	
go
</code>
</pre>

And now lets innocently query our UDF:

<pre><code class="prettyprint lang-sql">
select [value] 
	from dbo.udf_get_only_numerics ()
    where attribute = 'B1' 
    and cast([value] as int) > 50
go
</code>
</pre>

Since we&#8217;re selecting from the UDF, we&#8217;re guaranteed that we&#8217;re only going to see values that have the <tt>is_numeric</tt> column value 1, so the <tt>cast([value] as int)</tt> must always succeed, right? Wrong, we&#8217;ll get a conversion exception:

<pre><code class="prettyprint lang-sql">
&lt;span style="color:red">Msg 245, Level 16, State 1, Line 3
Conversion failed when converting the varchar value 'Gotch ya' to data type int.&lt;/span>
</code>
</pre>

This UDF was originally posted on <a href="http://www.sqlservercentral.com/articles/Tally+Table/72993/" target="_blank">Tally OH! An Improved SQL 8K “CSV Splitter” Function</a> article on sqlservercentral.com