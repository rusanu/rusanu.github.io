---
id: 1680
title: Case Sensitive collation sort order
date: 2012-11-23T01:15:30+00:00
author: remus
layout: post
guid: /?p=1680
permalink: /2012/11/23/case-sensitive-collation-sort-order/
categories:
  - Tutorials
tags:
  - case sensitive
  - collation
  - sort
  - sql
  - sql server
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

<!--more-->

<div>
  <div style="display: block; float: left; margin-left:  50px;">
    <a href="/wp-content/uploads/2012/11/Variant1.png"><img src="/wp-content/uploads/2012/11/Variant1.png" alt="" title="Variant1" width="165" height="130" class="alignleft size-full wp-image-1683" /></a>
  </div>
  
  <div style="display: block; float: right; margin-right:  50px;">
    <a href="/wp-content/uploads/2012/11/Variant2.png"><img src="/wp-content/uploads/2012/11/Variant2.png" alt="" title="Variant2" width="158" height="129" class="alignright size-full wp-image-1684" /></a>
  </div>
  
  <p>
    </div> 
    
    <p style="clear: both">
      Which one is correct? A programmer will chose the first order: <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt>, <tt>'A 2'</tt>. Because if one would implement a string comparison routine it would compare character by character until a difference is encountered, and <tt>'a'</tt> sorts ahead of <tt>'A'</tt>. But this answer is wrong. The correct sort order is <tt>'a 1'</tt>, <tt>'A 1'</tt>, <tt>'a 2'</tt>, <tt>'A 2'</tt>! And if you ran the query in SQL Server you certainly got the second output. But look again at the sort order and focus on just the first character:
    </p>
    
    <p>
      <a href="/wp-content/uploads/2012/11/Variant3.png"><img src="/wp-content/uploads/2012/11/Variant3.png" alt="" title="Variant3" width="158" height="129" class="alignleft size-full wp-image-1690" /></a>
    </p>
    
    <p class="callout float-right">
      By default, the <a href="http://www.unicode.org/reports/tr10/#Scope" target="_blank">algorithm</a> makes use of three fully-customizable levels. For the Latin script, these levels correspond roughly to: alphabetic ordering, diacritic ordering, case ordering
    </p>
    
    <p>
      So, in a case sensitive collation, is <tt>'a'</tt> ahead or after <tt>'A'</tt> in the sort order? The images shows them actually interleaved, is <tt>'a'</tt>, <tt>'A'</tt>, <tt>'a'</tt>, <tt>'A'</tt>. What&#8217;s going on? The answer is that collation sort order is a little more nuanced that just comparing characters until a difference is encountered. This is described in the <a href="http://www.unicode.org/reports/tr10">Unicode Technical Standard #10: UNICODE COLLATION ALGORITHM</a>. And yes, the same algorithm is applied for non-Unicode types (VARCHAR) too. The algorithm actually gives different weight to character differences and case differences, a difference in alphabetic order is more important than one in case order. To compare the sort order of two strings the algorithm is more like the following:
    </p>
    
    <ul>
      <li>
        Compare every character in case <b>insensitive</b>, accent <b>insensitive</b> manner. If a difference is found, this decides the sort order. If no difference is found, continue.
      </li>
      <li>
        Compare every character in case <b>insensitive</b>, accent <b>sensitive</b> manner. If a difference is found, this decides the sort order. If no difference is found, continue.
      </li>
      <li>
        Compare every character in case <b>sensitive</b> manner (we already know from the step above there is no accent difference). If a difference is found, this decides the sort order. If no difference is found the strings are equal.
      </li>
    </ul>
    
    <p>
      Needless to say the real algorithm does not need to traverse the strings 3 times, but the logic is equivalent to above. And remember that when the strings have different lengths then the comparison expands the shorter string with spaces and compares up to the length of the longest string. Combined with the case sensitivity rules this gives to a somewhat surprising result when using an inequality in a <tt>WHERE</tt> clause:
    </p>
    
    <pre><code class="prettyprint lang-sql">
select [col]
from [test]
where [col] > 'A'
order by [col];
</code></pre>
    
    <p>
      <a href="/wp-content/uploads/2012/11/Variant2.png"><img src="/wp-content/uploads/2012/11/Variant2.png" alt="" title="Variant2" width="158" height="129" class="alignright size-full wp-image-1684" /></a>
    </p>
    
    <p>
      That&#8217;s right, we got back all 4 rows, including those that start with <tt>'a'</tt>. This surprises some, but is the correct result. <tt>'a&nbsp;1'</tt> should be in the result, even though <tt>'a'</tt> is < <tt>'A'</tt>. If you follow the algorithm above: first we expand the shorter string with spaces, so the comparison is between <tt>'a&nbsp;1'</tt> and <tt>'A&nbsp;&nbsp;'</tt>. Then we do the first pass comparison, which is only alphabetic order, case insensitive and accent insensitive, character by character: <tt>'a'</tt> and <tt>'A'</tt> are equal, <tt>'&nbsp;'</tt> and <tt>'&nbsp;'</tt> are equal, but <tt>'1'</tt> is > <tt>'&nbsp;'</tt>. The comparison stops, we found a alphabetic order difference so <tt>'a&nbsp;1'</tt> > <tt>'A&nbsp;&nbsp;'</tt>, the row qualifies and is included in the result. Ditto for <tt>'a&nbsp;2'</tt>.
    </p>