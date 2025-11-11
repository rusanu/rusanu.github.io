---
id: 1657
title: Handling exceptions that occur during the RECEIVE statement in activated procedures
date: 2012-10-15T04:43:18+00:00
author: remus
layout: post
guid: /?p=1657
permalink: /2012/10/15/handling-exceptions-that-occur-during-the-receive-statement-in-activated-procedures/
categories:
  - Tutorials
tags:
  - error 9617
  - error handling
  - service broker
  - sql server
  - The service queue is currently disabled
  - transact-sql
---
The typical SQL Server activation procedure is contains a <tt>WHILE (1=1)</tt> loop and exit conditions based on checking <tt>@@ROWCOUNT</tt>. Error handling is done via a <tt>BEGIN TRY ... BEGIN CATCH</tt> block. This pattern is present in many Service Broker articles on the web, including this web site, in books and in Microsoft samples:

<!--more-->

  

<code class="prettyprint lang-sql linenums">
&lt;pre>
create procedure [&lt;procedure name&gt;]
as
declare @dialog_handle uniqueidentifier
 , @message_type_name sysname
 , @message_body varbinary(max);
set nocount on;

while(1=1)
begin 
 begin transaction;
 begin try;
  receive top(1) 
   @dialog_handle = conversation_handle
   , @message_type_name = message_type_name
   , @message_body = message_body
  from [&lt;queue name&gt;];
  if @@rowcount = 0
  begin
   rollback;
   break;
  end
  if @message_type_name = N'&lt;my message type&gt;'
  begin
   -- process the message here
                        ...
  end
  else if @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
     or @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'
  begin
   end conversation @dialog_handle;
  end
  commit transaction;
 end try
 begin catch
  declare @error_number int = ERROR_NUMBER()
   , @error_message nvarchar(4000) = ERROR_MESSAGE()
   , @xact_state int = XACT_STATE();
  if @xact_state = -1 or @xact_state = 1
  begin
   rollback;
  end
  -- log the error here
               ....
 end catch
end
go
&lt;/pre>
&lt;p></code>

This patter though contains a problem: it will handle very poorly a disabled queue, and hence it will handle very poorly poison messages.

## Error 9617 The service queue &#8220;&#8230;&#8221; is currently disabled

<!--more-->

Attempting to <tt>RECEIVE</tt> from a disabled queue will raise error 9617 but will not interrupt the batch. The above procedure error handling will handle the exception, eventually logging it somewhere, and then it will resume and loop again, hitting again error 9617. Your ERRORLOG file will likely grow quickly, the CPU will be busy due to the tight loop, all to no avail. Furthermore the constant executing RECEIVE and the activated procedure may even prevent you from re-enabling the queue as is being locked. The only way to stop such a runaway procedure is to [<TT>KILL it</TT>](http://msdn.microsoft.com/en-us/library/ms173730.aspx).

<p class="callout float-left">
  Separate the exception handling of the RECEIVE statement
</p>

You can try several solutions. You could check for the queue state before issuing the <tt>RECEIVE</tt>. You could special case error 9617 in the <tt>CATCH</tt> block. But if error 9617 requires special handling, are you sure you have covered all the special cases? I would recommend a different approach. What happens here is that your error handling does not differentiate between an exception that occurred in the <tt>RECEIVE</tt> statement itself and an exception that occurred in the processing of the messages returned by <tt>RECEIVE</tt>. Differentiating this two cases would allow for separate handling: if the <tt>RECEIVE</tt> statement itself has raised an exception then is better to abandon and exit the loop. If the exception occurred in the message processing then, after the exception is handled based on business specific rules, it is OK to dequeue more messages and issue <tt>RECEIVE</tt> again:  

<code class="prettyprint lang-sql linenums">
&lt;pre>
create procedure [&lt;procedure name&gt;]
as
declare @dialog_handle uniqueidentifier
 , @message_type_name sysname
 , @message_body varbinary(max)
 , @error_number int
 , @error_message nvarchar(4000)
 , @xact_state int;
set nocount on;

while(1=1)
begin 
 set  @dialog_handle = null;
 begin transaction;
 begin try;
  receive top(1) 
   @dialog_handle = conversation_handle
   , @message_type_name = message_type_name
   , @message_body = message_body
  from [&lt;queue name&gt;];
 end try 
begin catch
  -- this catch block handles errors in the RECEIVE statement
  set @error_number = ERROR_NUMBER();
  set @error_message = ERROR_MESSAGE();
  set @xact_state = XACT_STATE();
  if @xact_state = -1 or @xact_state = 1
  begin
   rollback;
  end
  -- log the error here
  ...
  break;
end catch
if @dialog_handle is null
begin
  rollback;
  break;
end
begin try
  if @message_type_name = N'&lt;my message type&gt;'
  begin
   -- process the message here
   ...
  end
  else if @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
     or @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'
  begin
   end conversation @dialog_handle;
  end
  commit transaction;
 end try
 begin catch
  -- this catch block handles errors in the message processing
  set @error_number = ERROR_NUMBER();
  set @error_message = ERROR_MESSAGE();
  set @xact_state = XACT_STATE();
  if @xact_state = -1 or @xact_state = 1
  begin
   rollback;
  end
  -- log the error here
 ... 
 end catch
end 
go
&lt;/pre>
&lt;p></code>

## Why loop in the first place?

The idea of the loop inside the activated stored procedure is that once activated, a procedure should <tt>RECEIVE</tt> until it drains the message queue and only then exit. But here is the deal: the [Queue Monitor](/2008/08/03/understanding-queue-monitors/) that launched the activated procedure already loops for you! It will keep calling the activated procedure until the procedure issues a <tt>RECEIVE</tt> statement that returns no rows. So you can get rid of the <tt>WHILE (1=1)</tt> loop, which simplifies the procedure and make sit more robust. If we do so, we no longer need to distinguish between errors that occurred in the <tt>RECEIVE</tt> statement vs. errors that occur in the message processing:


<code class="prettyprint lang-sql linenums">
&lt;pre>
create procedure [&lt;procedure name&gt;]
as
declare @dialog_handle uniqueidentifier
 , @message_type_name sysname
 , @message_body varbinary(max)
 , @error_number int
 , @error_message nvarchar(4000)
 , @xact_state int;
set nocount on;
 begin transaction;
 begin try;
  receive top(1) 
   @dialog_handle = conversation_handle
   , @message_type_name = message_type_name
   , @message_body = message_body
  from [&lt;queue name&gt;];

  if @message_type_name = N'&lt;my message type&gt;'
  begin
   -- process the message here
   ...
  end
  else if @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
     or @message_type_name = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'
  begin
   end conversation @dialog_handle;
  end
  commit transaction;
 end try
 begin catch
  set @error_number = ERROR_NUMBER();
  set @error_message = ERROR_MESSAGE();
  set @xact_state = XACT_STATE();
  if @xact_state = -1 or @xact_state = 1
  begin
   rollback;
  end
  -- log the error here
 end catch
go
&lt;/pre>
&lt;p></code>

Notice there is no longer need for <tt>@@ROWCOUNT</tt> check, that is implicit in the testing of <tt>@message_type_name</tt> because it will be NULL if no message was received. There is only one <tt>BEGIN TRY ... BEGIN CATCH ...</tt> block, which makes for an easier to comprehend, test and present procedure. And with the <tt>WHILE (1=1)</tt> loop eliminated there are fewer chances for a bug causing an infinite loop, as it was the case with error 9617. 

# Deploying blog samples in production

Over time I learned that a lot of times readers deploy in production the code samples straight from the blogs they read. You have to keep in mind that a blog example is written first and foremost to illustrate the point of the blog post and not to handle a real life production environment. In my production code there must be error handling, exception logging, performance measurement instrumentation in place (with run-time knobs to enable or disable the instrumentation, or to dial the level of detail up or down) and so on and so forth. Also in a real production case I would try to leverage batch processing of messages as described in [Writing Service Broker Procedures](/2006/10/16/writing-service-broker-procedures/). When processing batch messages using a cursor the individual message processing is an excellent candidate for using a transaction savepoint, in the manner described in [Exception Handling and Nested Transactions](/2009/06/11/exception-handling-and-nested-transactions/). Use your judgement and remember that this code here is a sample, only provided to guide you in the right direction.