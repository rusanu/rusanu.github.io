---
id: 557
title: On SQL Server boolean operator short-circuit
date: 2009-09-13T12:36:38+00:00
author: remus
layout: post
guid: /?p=557
permalink: /2009/09/13/on-sql-server-boolean-operator-short-circuit/
categories:
  - Tutorials
tags:
  - boolean
  - evaluation
  - expression
  - short-circuit
  - sql
  - tsql
---
Recently I had several discussions all circling around the short-circuit of boolean expressions in Transact-SQL queries. Many developers that come from an imperative language background like C are relying on boolean short-circuit to occur when SQL queries are executed. Often they take this expectation to extreme and the correctness of the result is actually relying on the short-circuit to occur:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'Will&nbsp;not&nbsp;divide&nbsp;by&nbsp;zero!'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">1&nbsp;&lt;/span>&lt;span style="color:Gray">or&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">/&lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">0&lt;/span>
&lt;/pre>
&lt;p></code>

In the SQL snippet above the expression on the right side of the OR operator would cause a division by zero if ever evaluated. Yet the query executes fine and the successful result is seen as proof that operator short-circuit does happen! Well, is that all there is? Of course not. An <a href="http://en.wikipedia.org/wiki/Universal_quantification" target="_blank">universal quantification</a> cannot be demonstrated with an example. But it can be proven false with one single counter example!

Luckily I have two aces on my sleeve: for one I know how the Query Optimizer works. Second, I&#8217;ve stayed close enough to Microsoft CSS front lines for 6 months to see actual cases pouring in from developers bitten by the short-circuit assumption. Here is my counter-example case:

<pre><code class="prettyprint lang-sql linenums">
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
end 
go 

-- insert a 'trap' 
insert into eav (attribute, is_numeric, [value]) values ('B1', 0, 'Gotch ya'); 
go
 
-- select the 'trap' value 
select [value] from eav 
	where      attribute = 'B1'      
	and is_numeric = 1      
	and cast([value] as int) > 50 
go 

Msg 245, Level 16, State 1, Line 3
Conversion failed when converting the varchar value 'Gotch ya' to data type int.
</code>
</pre>

This happens on SQL Server 2005 SP2. Clearly, the conversion **does** occur even though the value is marked as &#8216;not numeric&#8217;. Whats going on here? To better understand, lets insert a known value that can be converted and then run the same query again and look at the execution plan:

<pre><code class="prettyprint lang-sql linenums">
insert into eav (attribute, is_numeric, [value]) values ('B2', 1, 65); 
go 

select [value] from eav 
	where      attribute = 'B2'      
	and is_numeric = 1      
	and cast([value] as int) > 50;
go  
</code>
</pre>

<div id="attachment_559" style="width: 510px" class="wp-caption alignnone">
  <a href="/wp-content/uploads/2009/09/short-circuit.png"><img src="/wp-content/uploads/2009/09/short-circuit.png" alt="boolean short-circuit counter example query plan" title="short-circuit" width="500" height="123" class="size-full wp-image-559" /></a>
  
  <p class="wp-caption-text">
    boolean short-circuit counter example query plan
  </p>
</div>

Looking at the plan we can see how the query is actually evaluated: seek on the non-clustered index for the attribute &#8216;B2&#8217;, project the &#8216;value&#8217;, filter for the value predicate &#8216;cast([value] as int)>50&#8217; _then_ perform a nested join to look up the &#8216;is_boolean&#8217; in the clustered index! So **the right side of the AND operator is evaluated first**. Q.E.D.

Is this a bug? Of course not. SQL is a declarative language, the query optimizer is free to choose any execution path that provide the requested result. Boolean operator short-circuit is **NOT GUARANTEED**. My query has set up a trap for the query optimizer, by providing a tempting execution path using the non-clustered index. For my example to work I had to set up a large table and enough distinct values of &#8216;attribute&#8217; so that the optimizer would see the non-clustered index access followed by bookmark look up as a better plan than a clustered scan. And it is, by all means a better plan. But then I placed my trap: by adding the &#8216;value&#8217; as an included column in the non-clustered index, I give the optimizer a too sweet to resists opportunity to evaluate the filter predicate on the &#8216;value&#8217; column _before_ it evaluates the filter predicate on the &#8216;is_numeric&#8217; column, thus forcing the break on the short-circuit assumption.