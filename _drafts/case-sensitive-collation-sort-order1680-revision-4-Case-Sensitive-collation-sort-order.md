---
id: 1686
title: Case Sensitive collation sort order
date: 2012-11-22T23:50:41+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/11/22/1680-revision-4/
permalink: /2012/11/22/1680-revision-4/
---
A recent inquiry from one of our front line CSS engineers had me look into how case sensitive collations decide the sort order. Consider a simple question like _How should the values <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt> and <tt>'A 2'</tt> sort?_

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

Here are two possible outputs:

[<img src="http://rusanu.com/wp-content/uploads/2012/11/Variant1.png" alt="" title="Variant1" width="165" height="130" class="alignleft size-full wp-image-1683" />](http://rusanu.com/wp-content/uploads/2012/11/Variant1.png)  
[<img src="http://rusanu.com/wp-content/uploads/2012/11/Variant2.png" alt="" title="Variant2" width="158" height="129" class="alignright size-full wp-image-1684" />](http://rusanu.com/wp-content/uploads/2012/11/Variant2.png)