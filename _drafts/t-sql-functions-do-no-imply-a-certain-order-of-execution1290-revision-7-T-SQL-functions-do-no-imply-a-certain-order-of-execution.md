---
id: 1298
title: T-SQL functions do no imply a certain order of execution
date: 2011-08-10T01:44:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/10/1290-revision-7/
permalink: /2011/08/10/1290-revision-7/
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
</code></pre>

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
</code></pre>

And now lets innocently query our UDF:

<pre><code class="prettyprint lang-sql">
select [value] 
	from dbo.udf_get_only_numerics ()
    where attribute = 'B1' 
    and cast([value] as int) > 50
go
</code></pre>

Since we&#8217;re selecting from the UDF, we&#8217;re guaranteed that we&#8217;re only going to see values that have the <tt>is_numeric</tt> column value 1, so the <tt>cast([value] as int)</tt> must always succeed, right? Wrong, we&#8217;ll get a conversion exception:

<pre><code class="prettyprint lang-sql">
&lt;span style="color:red">Msg 245, Level 16, State 1, Line 3
Conversion failed when converting the varchar value 'Gotch ya' to data type int.&lt;/span>
</code></pre>

So what&#8217;s going on? Lets insert a known value and inspect the execution plan:

<pre>&lt;/code>
insert into eav (attribute, is_numeric, [value])
values ('B2', 1, 65);
go

select [value] 
	from dbo.udf_get_only_numerics ()
    where attribute = 'B2' 
    and cast([value] as int) > 50
go
&lt;/code></pre>

[<img src="http://rusanu.com/wp-content/uploads/2011/08/udf-eav-plan.png" alt="" title="udf-eav-plan" width="600" height="288" class="aligncenter size-full wp-image-1295" />](http://rusanu.com/wp-content/uploads/2011/08/udf-eav-plan.png)

Looking at the plan we see that, just as in my boolean short-circuit example, the presence of an index on <tt>attribute</tt> that includes the <tt>value</tt> column but not the <tt>is_numeric</tt> was too good opportunity to pass for the optimizer. And this plan will evaluate the <tt>cast([value]as int)</tt> expression **before** it evaluate the WHERE clause inside the UDF. That&#8217;s right, the WHERE clause of the UDF has moved in the execution plan _above_ the WHERE clause of the query using the UDF. The naive assumption that the function definition posses some sort of barrier for execution was proven wrong.

So how does this explains the failure seen in the StackOverflow question? The UDF, which was originally posted on <a href="http://www.sqlservercentral.com/articles/Tally+Table/72993/" target="_blank">Tally OH! An Improved SQL 8K “CSV Splitter” Function</a> article on sqlservercentral.com, contains WHERE clauses to that in effect restricts the output to only complete tokens. w/o the WHERE clause it would return non-complete token. We can easily test this, lets remove the WHERE clause and use the same input as the test in the StackOverflow question:

<pre><code class="prettyprint lang-sql">
CREATE FUNCTION dbo.DelimitedSplit8K_noWhere
--===== Define I/O parameters
        (@pString VARCHAR(8000), @pDelimiter CHAR(1))
RETURNS TABLE WITH SCHEMABINDING AS
 RETURN
--===== "Inline" CTE Driven "Tally Table" produces values from 0 up to 10,000...
     -- enough to cover VARCHAR(8000)
 WITH E1(N) AS (
   SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL 
   SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL 
   SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1 UNION ALL SELECT 1
                ),                          --10E+1 or 10 rows
       E2(N) AS (SELECT 1 FROM E1 a, E1 b), --10E+2 or 100 rows
       E4(N) AS (SELECT 1 FROM E2 a, E2 b), --10E+4 or 10,000 rows max
 cteTally(N) AS (
                 SELECT 0 UNION ALL
                 SELECT TOP (DATALENGTH(ISNULL(@pString,1))) 
                   ROW_NUMBER() OVER (ORDER BY (SELECT NULL)) FROM E4
                ),
cteStart(N1) AS (
                 SELECT t.N+1
                   FROM cteTally t
                )
 SELECT ItemNumber = ROW_NUMBER() OVER(ORDER BY s.N1),
        Item  = SUBSTRING(@pString,s.N1,
                ISNULL(NULLIF(CHARINDEX(@pDelimiter,@pString,s.N1),0)-s.N1,8000))
   FROM cteStart s;
</code></pre>

When we run this the output contains not only the two token in the input, but also every substring of these tokens:

<pre><code class="prettyprint lang-sql">
-- Test string that will be split into table in the DelimitedSplit8k function
declare @temp varchar(max) = 
     '918E809E-EA7A-44B5-B230-776C42594D91,6F8DBB54-5159-4C22-9B0A-7842464360A5'
select Item from dbo.DelimitedSplit8K_nowhere(@temp, ',');
go
Item
-----------------------------------------
918E809E-EA7A-44B5-B230-776C42594D91
18E809E-EA7A-44B5-B230-776C42594D91
8E809E-EA7A-44B5-B230-776C42594D91
E809E-EA7A-44B5-B230-776C42594D91
809E-EA7A-44B5-B230-776C42594D91
09E-EA7A-44B5-B230-776C42594D91
9E-EA7A-44B5-B230-776C42594D91
E-EA7A-44B5-B230-776C42594D91
-EA7A-44B5-B230-776C42594D91
...
</code></pre>

By now the explanation to the issue seen in the StackExchange question should be obvious: just as in my example, the WHERE clause was pulled out from the function and placed in the generated plan _above_ the <tt>cast(Item as uniqueidentifier)</tt> expression. During execution SQL Server is asked to convert the string &#8217;18E809E-EA7A-44B5-B230-776C42594D91&#8242; to a uniqueidentifier and, understandably, it fails.

Do no make assumptions about order of executions when using inline table UDFs. Q.E.D.