---
id: 1696
title: Case Sensitive collation sort order
date: 2012-11-23T00:25:02+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/11/23/1680-revision-13/
permalink: /2012/11/23/1680-revision-13/
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
    
    <p style="clear: both">
      Which one is correct? A programmer will chose the first order: <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt>, <tt>'A 2'</tt>. Because if one would implement a string comparison routine it would compare character by character until a difference is encountered, and <tt>'a'</tt> sorts ahead of <tt>'A'</tt>. But this answer is wrong. The correct sort order is <tt>'a 1'</tt>, <tt>'A 1'</tt>, <tt>'a 2'</tt>, <tt>'A 2'</tt>! And if you ran the query in SQL Server you certainly got the second output. But look again at the sort order and focus on just the first character:
    </p>
    
    <p>
      <a href="http://rusanu.com/wp-content/uploads/2012/11/Variant3.png"><img src="http://rusanu.com/wp-content/uploads/2012/11/Variant3.png" alt="" title="Variant3" width="158" height="129" class="alignleft size-full wp-image-1690" /></a>
    </p>
    
    <p class="callout float-right">
      By default, the <a href="http://www.unicode.org/reports/tr10/#Scope">algorithm</a> makes use of three fully-customizable levels. For the Latin script, these levels correspond roughly to: alphabetic ordering, diacritic ordering, case ordering
    </p>
    
    <p>
      So, in a case sensitive collation, is <tt>'a'</tt> ahead or after <tt>'A'</tt> in the sort order? The images shows them actually interleaved, is <tt>'a'</tt>, <tt>'A'</tt>, <tt>'a'</tt>, <tt>'A'</tt>. What&#8217;s going on? The answer is that collation sort order is a little more nuanced that just comparing characters until a difference is encountered.
    </p>