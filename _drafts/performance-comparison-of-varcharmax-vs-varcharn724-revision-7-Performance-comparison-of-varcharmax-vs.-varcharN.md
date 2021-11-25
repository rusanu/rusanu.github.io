---
id: 731
title: Performance comparison of varchar(max) vs. varchar(N)
date: 2010-03-22T20:59:59+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/22/724-revision-7/
permalink: /2010/03/22/724-revision-7/
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

The next text does a simple string comparison of a VARCHAR(MAX) field vs. a VARCHAR(8000) field. The actual value stored is identical in both cases, a simple 3 letter string &#8216;abc&#8217;. The data access does not retrieve the string value, it simply compares its value against a WHERE clause predicate:


<code class="prettyprint lang-sql">
&lt;pre>
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">table&lt;/span>&lt;span style="color:Black">&nbsp;test&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;id&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">identity&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">primary&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">key&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;x&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;test&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'abc'&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@x&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;0&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">();&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">while&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">&lt;&lt;/span>&lt;span style="color:Black">&nbsp;100000&lt;br />
&lt;/span>&lt;span style="color:Blue">begin&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@id&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@id&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;id&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;test&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;id&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Gray">and&lt;/span>&lt;span style="color:Black">&nbsp;x&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'abc'&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">+&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;br />
select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">datediff&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">ms&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@startTime&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">());&lt;/span>
&lt;/pre>
&lt;p></code>

Loop execution time for VARCHAR(8000): 3.1 seconds. For VARCHAR(MAX): 3.6 seconds.

Furthermore, if we change the WHERE predicate to compare against a max type variable instead of the literal &#8216;<span style="color:red">abc</span>&#8216; the loop time increases to 3.9 seconds.

### Conclusion

The code path that handles the MAX types (varchar, nvarchar and varbinary) is different from the code path that handles their equivalent non=max length types. The non-max types can internally be represented as an ordinary pointer-and-length structure. But the max types cannot be stored internally as a contiguous memory area, since they can possibly grow up to 2Gb. So they have to be represented by a streaming interface, similar to COM&#8217;s <a href="http://msdn.microsoft.com/en-us/library/aa380034%28VS.85%29.aspx" ref="nofollow" target="_blank">IStream</a>. This carries over to every operation that involves the max types, including simple assignment and comparison, since these operations are more complicated over a streaming interface. The biggest impact is visible in the code that allocates and assign max-type variables (my first test), but the impact is visible on every operation.

The impact is reasonable on most operations with the max type operations being about 10% slower. So is this something to worry about, or even worth blogging about, after all? Actually, in a recent project of mine I had to change a column type from varchar(max) to varchar(5000) to alleviate the impact of max-types performance. The code happened to be on a very very performance critical path and testing clearly showed the impact was not only measurable, was quite actually quite serious. The processing throughput increased by about 25% just from this minor change in my case.

Although much discussion exists around the storage of max types vs. the storage of non-max types and the similarities and differences between them (eg. in-row vs. out-of-row vs. row-overflow storage), there is also a more fundamental aspect that differentiate these types: SQL Server internal code differences. These differences manifest themselves even when the storage of the max type is optimized in-row