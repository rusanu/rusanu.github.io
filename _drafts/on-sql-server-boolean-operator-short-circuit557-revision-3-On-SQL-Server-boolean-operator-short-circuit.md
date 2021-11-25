---
id: 561
title: On SQL Server boolean operator short-circuit
date: 2009-09-13T12:29:42+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/09/13/557-revision-3/
permalink: /2009/09/13/557-revision-3/
---
Recently I had several discussions all circling around the short-circuit of boolean expressions in Transact-SQL queries. Many developers that come from an imperative language background like C are relying on boolean short-circuit to occur when SQL queries are executed. Often they take this expectation to extreme and the correctness of the result is actually relying on the short-circuit to occur:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'Will&nbsp;not&nbsp;divide&nbsp;by&nbsp;zero!'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">1&nbsp;&lt;/span>&lt;span style="color:Gray">or&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">/&lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">0&lt;/span>
&lt;/pre>
&lt;p></code>

In the SQL snippet above the expression on the left side of the OR operator would cause a division by zero if ever evaluated. Yet the query executes fine and the successful result is result, proof that operator short-circuit does happen! Well, is that all there is? Of course not. One positive example does not demonstrate that the short-circuit will happen in _all_ cases, does it? In other world, an <a href="http://en.wikipedia.org/wiki/Universal_quantification" target="_blank">universal quantification</a> cannot be demonstrated with an example. But it can be proven false with one single counter example!

Luckily I have two aces on my sleeve: for one I know how the Query Optimizer works. Second, I&#8217;ve stayed close enough to Microsoft CSS front lines for 6 months to see actual cases pouring in with developers bitten by the short-circuit assumption. so here is my counter-example case:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">table&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;eav_id&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">identity&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">primary&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">key&lt;/span>&lt;span style="color:Gray">,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;attribute&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">50&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">not&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">null,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;is_numeric&nbsp;&lt;/span>&lt;span style="color:Blue">bit&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">not&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">null,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;[value]&nbsp;&lt;/span>&lt;span style="color:Blue">sql_variant&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">null);&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">index&lt;/span>&lt;span style="color:Black">&nbsp;eav_attribute&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Black">&nbsp;eav&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">attribute&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">include&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">[value]&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">--&nbsp;Fill&nbsp;the&nbsp;table&nbsp;with&nbsp;random&nbsp;values&lt;br />
&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;br />
declare&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;0&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">while&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">&lt;&lt;/span>&lt;span style="color:Black">&nbsp;100000&lt;br />
&lt;/span>&lt;span style="color:Blue">begin&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@attribute&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">50&lt;/span>&lt;span style="color:Gray">),&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@is_numeric&nbsp;&lt;/span>&lt;span style="color:Blue">bit&lt;/span>&lt;span style="color:Gray">,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@value&nbsp;&lt;/span>&lt;span style="color:Blue">sql_variant&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@attribute&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'A'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">+&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">rand&lt;/span>&lt;span style="color:Gray">()*&lt;/span>&lt;span style="color:Black">1000&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">3&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@is_numeric&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">case&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">when&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">rand&lt;/span>&lt;span style="color:Gray">()&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">&gt;&lt;/span>&lt;span style="color:Black">&nbsp;0.5&nbsp;&lt;/span>&lt;span style="color:Blue">then&lt;/span>&lt;span style="color:Black">&nbsp;1&nbsp;&lt;/span>&lt;span style="color:Blue">else&lt;/span>&lt;span style="color:Black">&nbsp;0&nbsp;&lt;/span>&lt;span style="color:Blue">end&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">if&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">@is_numeric&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@value&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">rand&lt;/span>&lt;span style="color:Gray">()&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">*&lt;/span>&lt;span style="color:Black">&nbsp;100&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">else&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@value&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'Lorem&nbsp;ipsum'&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">attribute&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;is_numeric&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;[value]&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@attribute&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@is_numeric&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@value&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;@i&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;@i&lt;/span>&lt;span style="color:Gray">+&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">--&nbsp;insert&nbsp;a&nbsp;'trap'&lt;br />
&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">attribute&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;is_numeric&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;[value]&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'B1'&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;0&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'Gotch&nbsp;ya'&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">GO&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">--&nbsp;select&nbsp;the&nbsp;'trap'&nbsp;value&lt;br />
&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;[value]&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;attribute&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'B1'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Gray">and&lt;/span>&lt;span style="color:Black">&nbsp;is_numeric&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;1&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Gray">and&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">[value]&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">&gt;&lt;/span>&lt;span style="color:Black">&nbsp;50&lt;br />
go&lt;/span>
&lt;/pre>
&lt;p></code>

<pre><span style="color: Red">Msg&nbsp;245,&nbsp;Level&nbsp;16,&nbsp;State&nbsp;1,&nbsp;Line&nbsp;3<br />
Conversion&nbsp;failed&nbsp;when&nbsp;converting&nbsp;the&nbsp;varchar&nbsp;value&nbsp;'Gotch&nbsp;ya'&nbsp;to&nbsp;data&nbsp;type&nbsp;int.</span>
</pre>

This happens on SQL Server 2005 SP2. Clearly, the conversion **does** occur even though the value is marked as &#8216;not numeric&#8217;. Whats going on here? To better understand, lets insert a known value that can be converted and then run the same query again and look at the execution plan:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">attribute&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;is_numeric&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;[value]&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'B2'&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;1&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;65&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;[value]&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;eav&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;attribute&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Red">'B2'&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Gray">and&lt;/span>&lt;span style="color:Black">&nbsp;is_numeric&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;1&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Gray">and&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">[value]&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">&gt;&lt;/span>&lt;span style="color:Black">&nbsp;50&lt;br />
go&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>  


<div id="attachment_559" style="width: 510px" class="wp-caption alignnone">
  <a href="http://test.rusanu.com/wp-content/uploads/2009/09/short-circuit.png"><img src="http://test.rusanu.com/wp-content/uploads/2009/09/short-circuit.png" alt="boolean short-circuit counter example query plan" title="short-circuit" width="500" height="123" class="size-full wp-image-559" /></a>
  
  <p class="wp-caption-text">
    boolean short-circuit counter example query plan
  </p>
</div>

Looking at the plan we can see how is the query actually evaluated: seek on the non-clustred index for the attribute &#8216;B2&#8217;, project the &#8216;value&#8217;, filter for the value predicate &#8216;cast([value] as int)>50&#8217; _then_ perform a nested join too look up the &#8216;is_boolean&#8217; in the clustered index! So the right side of the boolean expression is evaluated **first**. Q.E.D.

Is this a bug? Of course not. SQL is a declarative language, the query optimizer is free to choose any execution path that provide the requested result. Boolean operator short-circuit is **NOT GUARANTEED**. My query has set up a trap for the query optimizer, by providing a tempting execution path using the non-clustered index. For my example to work I had to set up a large table and enough distinct values of &#8216;attribute&#8217; so that the optimizer would see the non-clustered index access followed by bookmark look up as a better plan than a clustered scan. And it is, by all means a better plan. But then I placed my trap: by adding the &#8216;value&#8217; as an included column in the non-clustered index, I give the optimizer a too sweet to resists opportunity to evaluate the filter predicate on the &#8216;value&#8217; column _before_ it evaluates the filter predicate on the &#8216;is_numeric&#8217; column, thus forcing the break on the short-circuit assumption.