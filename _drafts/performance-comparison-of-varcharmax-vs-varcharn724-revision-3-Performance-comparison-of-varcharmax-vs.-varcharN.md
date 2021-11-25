---
id: 727
title: Performance comparison of varchar(max) vs. varchar(N)
date: 2010-03-17T23:26:13+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/17/724-revision-3/
permalink: /2010/03/17/724-revision-3/
---
The question of comparing the MAX types (VARCHAR, NVARCHAR, VARBINARY) with their non-max counterparts is often asked, but the answer usually gravitate around the storage differences. But I&#8217;d like to address the point that these types have inherent, intrinsic performance differences that are not driven by different storage characteristics. In other words, simply comparing and manipulating variables and columns in T-SQL can yield different performance when VARCHAR(MAX) is used vs. VARCHAR(N).

### Assignment

First comparing simple assignment, assign a value to a VARBINARY(8000) variable in a tight loop:


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

This script runs on my test server in 4.9 seconds on average. Simply changing the variable declaration to VARBINARY(MAX) changes the run time to an average of of 9.2 seconds.

### Comparison

Next I measured the performance of a simple comparison:


<code class="prettyprint lang-sql">
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">8000&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;0&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">();&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'abc'&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">while&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">&lt;&lt;/span>&lt;span style="color:Black">&nbsp;1000000&lt;br />
&lt;/span>&lt;span style="color:Blue">begin&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@y&nbsp;&lt;/span>&lt;span style="color:Blue">bit&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@y&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">case&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">when&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'abc'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">then&lt;/span>&lt;span style="color:Black">&nbsp;1&nbsp;&lt;/span>&lt;span style="color:Blue">else&lt;/span>&lt;span style="color:Black">&nbsp;0&nbsp;&lt;/span>&lt;span style="color:Blue">end&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">+&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;br />
select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">datediff&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">ms&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">());&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>

Average run time for VARCHAR(8000): 5.8 seconds. For VARCHAR(MAX): 6.5 seconds.

### Data Access