---
id: 2062
title: Asynchronous procedure execution
date: 2010-12-30T14:08:08+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/12/30/514-revision-22/
permalink: /2010/12/30/514-revision-22/
---
**Update:** a version of this sample that accepts parameters is available in the post [Passing Parameters to a Background Procedure](http://rusanu.com/2009/08/18/passing-parameters-to-a-background-procedure/)

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


<p>
  <code class="prettyprint lang-sql">&lt;/p>
  &lt;pre>
  &lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create procedure &lt;/span>&lt;span style="color:Black">usp_AsyncExecActivated&lt;br />
  &lt;/span>&lt;span style="color:Blue">as&lt;br />
  begin&lt;br />
      set nocount on&lt;/span>&lt;span style="color:Gray">;&lt;br />
      &lt;/span>&lt;span style="color:Blue">declare &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
          , &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Blue">xml&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@procedureName &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@startTime &lt;/span>&lt;span style="color:Blue">datetime&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@finishTime &lt;/span>&lt;span style="color:Blue">datetime&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorNumber &lt;/span>&lt;span style="color:Blue">int&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorMessage &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">)&lt;br />
          , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Blue">smallint&lt;br />
          &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;/span>&lt;span style="color:Gray">;&lt;br />
      &lt;br />
      &lt;/span>&lt;span style="color:Blue">begin transaction&lt;/span>&lt;span style="color:Gray">;&lt;br />
      &lt;/span>&lt;span style="color:Blue">begin try&lt;/span>&lt;span style="color:Gray">;&lt;br />
          &lt;/span>&lt;span style="color:Blue">receive top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">) &lt;br />
              &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_handle]&lt;br />
              &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[message_type_name]&lt;br />
              &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[message_body]&lt;br />
              &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">[AsyncExecQueue]&lt;/span>&lt;span style="color:Gray">;&lt;br />
          &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Gray">is not null)&lt;br />
          &lt;/span>&lt;span style="color:Blue">begin&lt;br />
              if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'DEFAULT'&lt;/span>&lt;span style="color:Gray">)&lt;br />
              &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                  &lt;/span>&lt;span style="color:Green">-- The DEFAULT message type is a procedure invocation.&lt;br />
                  -- Extract the name of the procedure from the message body.&lt;br />
                  --&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">CAST&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">as xml&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@procedureName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;br />
                      &lt;/span>&lt;span style="color:Red">'(//procedure/name)[1]'&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'sysname'&lt;/span>&lt;span style="color:Gray">);&lt;br />
  &lt;br />
                  &lt;/span>&lt;span style="color:Blue">save transaction &lt;/span>&lt;span style="color:Black">usp_AsyncExec_procedure&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@startTime &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">GETUTCDATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                  &lt;/span>&lt;span style="color:Blue">begin try&lt;br />
                      exec &lt;/span>&lt;span style="color:Black">@procedureName&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end try&lt;br />
                  begin catch&lt;br />
                  &lt;/span>&lt;span style="color:Green">-- This catch block tries to deal with failures of the procedure execution&lt;br />
                  -- If possible it rolls back to the savepoint created earlier, allowing&lt;br />
                  -- the activated procedure to continue. If the executed procedure &lt;br />
                  -- raises an error with severity 16 or higher, it will doom the transaction&lt;br />
                  -- and thus rollback the RECEIVE. Such case will be a poison message,&lt;br />
                  -- resulting in the queue disabling.&lt;br />
                  --&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@execErrorNumber &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_NUMBER&lt;/span>&lt;span style="color:Gray">(),&lt;br />
                      &lt;/span>&lt;span style="color:Black">@execErrorMessage &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_MESSAGE&lt;/span>&lt;span style="color:Gray">(),&lt;br />
                      &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">XACT_STATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                  &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= -&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
                  &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                      rollback&lt;/span>&lt;span style="color:Gray">;&lt;br />
                      &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Unrecoverable error in procedure %s: %i: %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">10&lt;/span>&lt;span style="color:Gray">,&lt;br />
                          &lt;/span>&lt;span style="color:Black">@procedureName&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorNumber&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorMessage&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end&lt;br />
                  else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
                  &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                      rollback transaction &lt;/span>&lt;span style="color:Black">usp_AsyncExec_procedure&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end&lt;br />
                  end catch&lt;br />
  &lt;br />
                  select &lt;/span>&lt;span style="color:Black">@finishTime &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">GETUTCDATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_id] &lt;br />
                      &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys.conversation_endpoints &lt;br />
                      &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[conversation_handle] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">is null)&lt;br />
                  &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                      raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Internal consistency error: conversation not found'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">20&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end&lt;br />
                  update &lt;/span>&lt;span style="color:Black">[AsyncExecResults] &lt;/span>&lt;span style="color:Blue">set&lt;br />
                      &lt;/span>&lt;span style="color:Black">[start_time] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@starttime&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[finish_time] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@finishTime&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_number] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@execErrorNumber&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_message] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@execErrorMessage&lt;br />
                      &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[token] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@token&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">0 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">@@ROWCOUNT&lt;/span>&lt;span style="color:Gray">)&lt;br />
                  &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                      raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Internal consistency error: token not found'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">30&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end&lt;br />
                  end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
              &lt;/span>&lt;span style="color:Blue">end &lt;br />
              else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'&lt;/span>&lt;span style="color:Gray">)&lt;br />
              &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                  end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
              &lt;/span>&lt;span style="color:Blue">end&lt;br />
              else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'&lt;/span>&lt;span style="color:Gray">)&lt;br />
              &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                  declare &lt;/span>&lt;span style="color:Black">@errorNumber &lt;/span>&lt;span style="color:Blue">int&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@errorMessage &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">4000&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">CAST&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">as xml&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Blue">with xmlnamespaces &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">DEFAULT &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'&lt;/span>&lt;span style="color:Gray">)&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@errorNumber &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'(/Error/Code)[1]'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'INT'&lt;/span>&lt;span style="color:Gray">),&lt;br />
                      &lt;/span>&lt;span style="color:Black">@errorMessage &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'(/Error/Description)[1]'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'NVARCHAR(4000)'&lt;/span>&lt;span style="color:Gray">);&lt;br />
                  &lt;/span>&lt;span style="color:Green">-- Update the request with the received error&lt;br />
                  &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_id] &lt;br />
                      &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys.conversation_endpoints &lt;br />
                      &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[conversation_handle] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">update &lt;/span>&lt;span style="color:Black">[AsyncExecResults] &lt;/span>&lt;span style="color:Blue">set&lt;br />
                      &lt;/span>&lt;span style="color:Black">[error_number] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@errorNumber&lt;br />
                      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_message] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@errorMessage&lt;br />
                      &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[token] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@token&lt;/span>&lt;span style="color:Gray">;&lt;br />
                  &lt;/span>&lt;span style="color:Blue">end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
               &lt;/span>&lt;span style="color:Blue">end&lt;br />
             else&lt;br />
             begin&lt;br />
                  raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Received unexpected message type: %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">50&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName&lt;/span>&lt;span style="color:Gray">);&lt;br />
             &lt;/span>&lt;span style="color:Blue">end&lt;br />
          end&lt;br />
          commit&lt;/span>&lt;span style="color:Gray">;&lt;br />
      &lt;/span>&lt;span style="color:Blue">end try&lt;br />
      begin catch&lt;br />
          declare &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Blue">int&lt;br />
              &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">);&lt;br />
          &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_NUMBER&lt;/span>&lt;span style="color:Gray">()&lt;br />
              , &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_MESSAGE&lt;/span>&lt;span style="color:Gray">()&lt;br />
              , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">XACT_STATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
          &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">&lt;&gt; &lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">)&lt;br />
          &lt;/span>&lt;span style="color:Blue">begin&lt;br />
              rollback&lt;/span>&lt;span style="color:Gray">;&lt;br />
          &lt;/span>&lt;span style="color:Blue">end&lt;/span>&lt;span style="color:Gray">;&lt;br />
          &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Error: %i, %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">60&lt;/span>&lt;span style="color:Gray">,  &lt;/span>&lt;span style="color:Black">@error&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">with &lt;/span>&lt;span style="color:Fuchsia">log&lt;/span>&lt;span style="color:Gray">;&lt;br />
      &lt;/span>&lt;span style="color:Blue">end catch&lt;br />
  end&lt;br />
  &lt;/span>&lt;span style="color:Black">go&lt;br />
  &lt;/span>
  &lt;/pre>
  &lt;p></code>
</p>


<p>
  To make the procedure activated we need to attach it to our service queue. This will ensure this procedure is run whenever a message arrives to our [AsyncExecService]:
</p>


<pre>
<span style="color: Black"></span><span style="color:Blue">alter</span><span style="color:Black">&nbsp;</span><span style="color:Blue">queue</span><span style="color:Black">&nbsp;[AsyncExecQueue]<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">with</span><span style="color:Black">&nbsp;activation&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;procedure_name&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;[usp_AsyncExecActivated]<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;max_queue_readers&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Blue">execute</span><span style="color:Black">&nbsp;</span><span style="color:Blue">as</span><span style="color:Black">&nbsp;</span><span style="color:Blue">owner<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Blue">status</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">);<br />
</span><span style="color:Black">go<br />
</span>
</pre>


<p>
  And finaly the last piece of the puzzle: the procedure that submits the message to invoke the desired asyncronous executed procedure. This procedure resturns an output parameter &#8216;token&#8217; than can be used to lookup the asynchronous execution result.
</p>


<pre>
<span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">procedure</span><span style="color:Black">&nbsp;[usp_AsyncExecInvoke]<br />
&nbsp;&nbsp;&nbsp;&nbsp;@procedureName&nbsp;</span><span style="color:Blue">sysname<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Blue">uniqueidentifier</span><span style="color:Black">&nbsp;</span><span style="color:Blue">output<br />
as<br />
begin<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@h&nbsp;</span><span style="color:Blue">uniqueidentifier<br />
</span><span style="color:Black">	&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@xmlBody&nbsp;</span><span style="color:Blue">xml<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Blue">int</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Black">	</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@trancount</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">else<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">save</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction</span><span style="color:Black">&nbsp;usp_AsyncExecInvoke</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">try<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">dialog</span><span style="color:Black">&nbsp;</span><span style="color:Blue">conversation</span><span style="color:Black">&nbsp;@h<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;</span><span style="color:Blue">service</span><span style="color:Black">&nbsp;[AsyncExecService]<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">to</span><span style="color:Black">&nbsp;</span><span style="color:Blue">service</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'AsyncExecService'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Red">'current&nbsp;database'<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">with</span><span style="color:Black">&nbsp;</span><span style="color:Blue">encryption</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">off</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;[conversation_id]<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;</span><span style="color:Green">sys.conversation_endpoints<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;[conversation_handle]&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@h</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@xmlBody&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@procedureName&nbsp;</span><span style="color:Blue">as</span><span style="color:Black">&nbsp;[name]<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">for</span><span style="color:Black">&nbsp;</span><span style="color:Blue">xml</span><span style="color:Black">&nbsp;</span><span style="color:Blue">path</span><span style="color:Gray">(</span><span style="color:Red">'procedure'</span><span style="color:Gray">),</span><span style="color:Black">&nbsp;</span><span style="color:Blue">type</span><span style="color:Gray">);<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">send</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Black">&nbsp;</span><span style="color:Blue">conversation</span><span style="color:Black">&nbsp;@h&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">@xmlBody</span><span style="color:Gray">);<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">insert</span><span style="color:Black">&nbsp;</span><span style="color:Blue">into</span><span style="color:Black">&nbsp;[AsyncExecResults]<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">[token]</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;[submit_time]</span><span style="color:Gray">)<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">values<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">@token</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">getutcdate</span><span style="color:Gray">());<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">commit</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">end</span><span style="color:Black">&nbsp;</span><span style="color:Blue">try<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">catch<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Blue">int<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@message&nbsp;</span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">)<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@xactState&nbsp;</span><span style="color:Blue">smallint</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">()<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@message&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">()<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@xactState&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xactState&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Gray">-</span><span style="color:Black">1<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">rollback</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xactState&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1&nbsp;</span><span style="color:Gray">and</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">rollback<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xactState&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1&nbsp;</span><span style="color:Gray">and</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">&gt;</span><span style="color:Black">&nbsp;0<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">rollback</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction</span><span style="color:Black">&nbsp;usp_my_procedure_name</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Black">N</span><span style="color:Red">'Error:&nbsp;%i,&nbsp;%s'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;16</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@error</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@message</span><span style="color:Gray">);<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">end</span><span style="color:Black">&nbsp;</span><span style="color:Blue">catch<br />
end<br />
</span><span style="color:Black">go</span>
</pre>


<p>
  To test our asynchronous execution infrastructure we create a test procedure and invoke it asynchronously. I will create two test procedures, one that simply waits for 5 seconds to simulate a &#8216;long&#8217; running procedure and one that produces intentionally a primary key violation, to simulate a fault in the asynchronously executed procedure:
</p>


<pre>
<span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">procedure</span><span style="color:Black">&nbsp;[usp_MyLongRunningProcedure]<br />
</span><span style="color:Blue">as<br />
begin<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">waitfor</span><span style="color:Black">&nbsp;</span><span style="color:Blue">delay</span><span style="color:Black">&nbsp;</span><span style="color:Red">'00:00:05'</span><span style="color:Gray">;<br />
</span><span style="color:Blue">end<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">procedure</span><span style="color:Black">&nbsp;[usp_MyFaultyProcedure]<br />
</span><span style="color:Blue">as<br />
begin<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@t&nbsp;</span><span style="color:Blue">table</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">id&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;</span><span style="color:Blue">primary</span><span style="color:Black">&nbsp;</span><span style="color:Blue">key</span><span style="color:Gray">);<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">insert</span><span style="color:Black">&nbsp;</span><span style="color:Blue">into</span><span style="color:Black">&nbsp;@t&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">id</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">values</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">);<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">insert</span><span style="color:Black">&nbsp;</span><span style="color:Blue">into</span><span style="color:Black">&nbsp;@t&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">id</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">values</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">);<br />
</span><span style="color:Blue">end<br />
</span><span style="color:Black">go<br />
<br />
<br />
</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;<br />
</span><span style="color:Blue">exec</span><span style="color:Black">&nbsp;usp_AsyncExecInvoke&nbsp;N</span><span style="color:Red">'usp_MyLongRunningProcedure'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Blue">output</span><span style="color:Gray">;<br />
</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;</span><span style="color:Gray">*</span><span style="color:Black">&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;[AsyncExecResults]&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;[token]&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@token</span><span style="color:Gray">;<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;<br />
</span><span style="color:Blue">exec</span><span style="color:Black">&nbsp;usp_AsyncExecInvoke&nbsp;N</span><span style="color:Red">'usp_MyFaultyProcedure'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@token&nbsp;</span><span style="color:Blue">output</span><span style="color:Gray">;<br />
</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;</span><span style="color:Gray">*</span><span style="color:Black">&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;[AsyncExecResults]&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;[token]&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@token</span><span style="color:Gray">;<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">waitfor</span><span style="color:Black">&nbsp;</span><span style="color:Blue">delay</span><span style="color:Black">&nbsp;</span><span style="color:Red">'00:00:10'</span><span style="color:Gray">;<br />
</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;</span><span style="color:Gray">*</span><span style="color:Black">&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;[AsyncExecResults]</span><span style="color:Gray">;<br />
</span><span style="color:Black">go</span>
</pre>


<h3>
  Activation Context
</h3>


<p>
  If you check the start time of the second asynchronosuly executed procedure you will notice that it started right after the first one finished. This is because we declare a <tt>max_queue_readers</tt> value of 1 when we set up activation on the queue. This restricts that at most one activated procedure to run at any time, effectively serializing all the asynchronously executed procedures. Whether this is desired or not depends a lot on the actual usage scenario. The limit can be increased as necessary.
</p>


<p>
  If you start playing around with this method of invoking procedures asynchronously you will notice that sometimes the asynchrnously executed procedure is misteriously denied access to other databases or to server scoped objects. When the same procedure is run manually from a query window in SSMS, it executes fine. This is caused by the EXECUTE AS context under which activation occurs. the details are explained in MSDN&#8217;s <a href="http://msdn.microsoft.com/en-us/library/ms188304.aspx" target="_blank">Extending Database Impersonation by Using EXECUTE AS</a> and myself I had covered this subject repeatedly in this blog. The best solution is to simply turn the trustworthy bit on on the database where the activated procedure runs. When this is not desired, or not allowed by your hosting environment, the solution is to code sign the activated procedure: <a href="http://rusanu.com/2006/03/01/signing-an-activated-procedure/">Signing an activated procedure</a>.
</p>


<p>
  Using Service Broker Activation to invoke procedures asynchronously may look daunting at beginning. It sure is significantly more complex than just calling BeginExecuteNonQuery. But what needs to be understood is that this is a <strong>reliable</strong> way to invoke the procedure. The client is free to disconnect as soon as it commited the call to <tt>usp_AsyncExecInvoke</tt>. The procedure invoked <i>will</i> run, even if the server is stopped and restarted, even if a mirroring or clustering failover occurs. The server may even crash and be completely rebuilt. As soon as the database is back online, that queue will activate and invoke the asynchronous execution. Such level of reliability is difficult, if not impossible, to guarantee by using a client process.
</p>