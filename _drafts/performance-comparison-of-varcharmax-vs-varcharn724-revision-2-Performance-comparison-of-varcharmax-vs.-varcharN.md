---
id: 726
title: Performance comparison of varchar(max) vs. varchar(N)
date: 2010-03-17T23:06:39+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/17/724-revision-2/
permalink: /2010/03/17/724-revision-2/
---
The question of comparing the MAX types (VARCHAR, NVARCHAR, VARBINARY) with their non-max counterparts is often asked, but the answer usually gravitate around the storage differences. But I&#8217;d like to address the point that these types have inherent, intrinsic performance differences that are not driven by different storage characteristics. In other words, simply comparing and manipulating variables and columns in T-SQL can yield different performance when VARCHAR(MAX) is used vs. VARCHAR(N).

First 


<code class="prettyprint lang-sql">
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">8000&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;0&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">();&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">while&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">&lt;&lt;/span>&lt;span style="color:Black">&nbsp;1000000&lt;br />
&lt;/span>&lt;span style="color:Blue">begin&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'abc'&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">+&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;br />
select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">datediff&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">ms&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">());&lt;/span>
&lt;/pre>
&lt;p></code>