---
id: 1682
title: Case Sensitive collation sort order
date: 2012-11-22T23:45:50+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/11/22/1680-revision-2/
permalink: /2012/11/22/1680-revision-2/
---
A recent inquiry from one of our front line CSS engineers had me look into how case sensitive collations decide the sort order. Consider a simple question like _How should the values <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt> and <tt>'A 2'</tt> sort?_

. 

<pre><code class="prettyprint lang-sql">
create table [test] (
	[col] varchar(10) 
		collate Latin1_General_CS_AS);
go

insert into [test] ([col]) values 
	('a 1'),
	('a 2'),
	('A 1'),
	('A 2');
go

select [col]
from [test]
order by [col];
</code></pre>