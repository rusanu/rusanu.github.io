---
id: 1689
title: Case Sensitive collation sort order
date: 2012-11-23T00:14:16+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/11/23/1680-revision-7/
permalink: /2012/11/23/1680-revision-7/
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

<div>
  <div style="display: block; float: left; margin-left:  50px;">
    <a href="http://rusanu.com/wp-content/uploads/2012/11/Variant1.png"><img src="http://rusanu.com/wp-content/uploads/2012/11/Variant1.png" alt="" title="Variant1" width="165" height="130" class="alignleft size-full wp-image-1683" /></a>
  </div>
  
  <div style="display: block; float: right; margin-right:  50px;">
    <a href="http://rusanu.com/wp-content/uploads/2012/11/Variant2.png"><img src="http://rusanu.com/wp-content/uploads/2012/11/Variant2.png" alt="" title="Variant2" width="158" height="129" class="alignright size-full wp-image-1684" /></a>
  </div>
  
  <p>
    </div> 
    
    <p>
      Which one is correct? The overwhelming number of programmers I talked with will chose the first order: <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt>, <tt>'A 2'</tt>. Because if one would implement a string comparison routine it would compare character by character until a difference is encountered, and <tt>'a'</tt> sorts ahead of <tt>'A'</tt>. But this answer is wrong. The correct sort order is <tt>'a 1'</tt>, <tt>'A 1'</tt>, <tt>'a 2'</tt>, <tt>'A 2'</tt>! And if you ran the query in SQL Server you certainly got the second output.
    </p>