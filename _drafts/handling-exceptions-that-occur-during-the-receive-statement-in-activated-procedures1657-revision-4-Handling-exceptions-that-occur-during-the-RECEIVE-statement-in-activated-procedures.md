---
id: 1662
title: Handling exceptions that occur during the RECEIVE statement in activated procedures
date: 2012-10-15T00:59:46+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/10/15/1657-revision-4/
permalink: /2012/10/15/1657-revision-4/
---
The typical SQL Server activation procedure is contains a <tt>WHILE (1=1)</tt> loop and exit conditions based on checking <tt>@@ROWCOUNT</tt>. Error handling is done via a <tt>BEGIN TRY ... BEGIN CATCH</tt> block. This pattern is present in many Service Broker articles on the web, including this web site, in books and in Microsoft samples:


<code language="SQL">
&lt;pre>
create procedure [&lt;procedure name>]
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
  from [&lt;queue name>];
  if @@rowcount = 0
  begin
   rollback;
   break;
  end
  if @message_type_name = N'&lt;my message type>'
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

Attempting to <tt>RECEIVE</tt> from a disabled queue will raise error 9617 but will not interrupt the batch. The above procedure error handling will handle the exception, eventually logging it somewhere, and then it will resume and loop again, hitting again error 9617. Your ERRORLOG file will likely grow quickly, the CPU will be busy due to the tight loop, all to no avail. Furthermore the constant executing RECEIVE and the activated procedure may even prevent you from re-enabling the queue as is being locked. The only way to stop such a runaway procedure is to [<TT>KILL it</TT>](http://msdn.microsoft.com/en-us/library/ms173730.aspx).

<p class="callout float-left">
  Separate the exception handling of the RECEIVE statement
</p>

You can try several solutions. You could check for the queue state before issuing the <tt>RECEIVE</tt>. You could special case error 9617 in the <tt>CATCH</tt> block

. But if error 9617 requires special handling, are you sure you have covered all the special cases? I would recommend a different approach.