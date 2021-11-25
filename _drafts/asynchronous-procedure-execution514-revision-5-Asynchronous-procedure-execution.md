---
id: 519
title: Asynchronous procedure execution
date: 2009-08-05T00:20:08+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/08/05/514-revision-5/
permalink: /2009/08/05/514-revision-5/
---
Recently an user on StackOverflow raised the question <a href="http://stackoverflow.com/questions/1229438/execute-a-stored-procedure-from-a-windows-form-asynchronously-and-then-disconnect" target="_blank">Execute a stored procedure from a windows form asynchronously and then disconnect?</a>. This is a known problem, how to invoke a long running procedure on SQL Server without constraining the client to wait for the procedure execution to terminate. Most times I&#8217;ve seen this question raised in the context of web applications when waiting for a result means delaying the response to the client browser. On Web apps the time constraint is even more drastic, the developer often desires to launch the procedure and immediately return the page even when the execution lasts only few seconds. The application will retrieve the execution result later, usually via an Ajax call driven by the returned page script.

Frankly I was a bit surprised to see that the responses gravitated either around the SqlClient asynchronous methods (BeginExecute&#8230;) or around having a dedicated process with the sole pupose of maintaining the client connection alive for the duration of the long running procedure.

This problem is perfectly addressed by Service Broker Activation. Since I wanted to preserve the solution for further reference, I decided to put it in as a blog entry, with additional comments. For many of you Service Broker aficionados that read my blog regularly, this article is not innovative as is mostly a rehash of well known techniques I&#8217;ve been talking about on forums for many years now.

I&#8217;m going to use a table to store the result of the procedure execution. In this version I&#8217;ll keep things simple by not allowing for any parameters to be passed to the procedure, nor collecting any execution result set data. So the table will only contain the procedure start time, the execution finish time and any error that occurred during the procedure execution:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">table</span><span style="color:Black">&nbsp;[AsyncExecResults]&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">	[token]&nbsp;</span><span style="color:Blue">uniqueidentifier</span><span style="color:Black">&nbsp;</span><span style="color:Blue">primary</span><span style="color:Black">&nbsp;</span><span style="color:Blue">key<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[submit_time]&nbsp;</span><span style="color:Blue">datetime</span><span style="color:Black">&nbsp;</span><span style="color:Gray">not</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[start_time]&nbsp;</span><span style="color:Blue">datetime</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[finish_time]&nbsp;</span><span style="color:Blue">datetime</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[error_number]	</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[error_message]&nbsp;</span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null);<br />
</span><span style="color:Black">go<br />
</pre>


<p>
  Next we&#8217;re going to create the service and queue we need. I will use one single service for both roles (initiator and target) and I won&#8217;t create an explicit contract, relying instead on the predefined DEFAULT contract:
</p>


<pre>
&lt;/span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">queue</span><span style="color:Black">&nbsp;[AsyncExecQueue]</span><span style="color:Gray">;<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">service</span><span style="color:Black">&nbsp;[AsyncExecService]&nbsp;</span><span style="color:Blue">on</span><span style="color:Black">&nbsp;</span><span style="color:Blue">queue</span><span style="color:Black">&nbsp;[AsyncExecQueue]&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">[DEFAULT]</span><span style="color:Gray">);<br />
</span><span style="color:Black">GO<br />
</span>
</pre>


<p>
  Next is the core of our asynchronous execution: the activated procedure. The procedure has to dequeue the message that specifies the user procedure, run the procedure and write the result in the results table. I will also deploy the error handling template I elaborated on my previous article <a href="http://rusanu.com/2009/06/11/exception-handling-and-nested-transactions/" target="_blank">Exception handling and nested transactions</a>:
</p>


<pre>

</pre>