---
id: 1659
title: Handling exceptions that occur during the RECEIVE statement in activated procedures
date: 2012-10-15T00:35:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/10/15/1657-revision/
permalink: /2012/10/15/1657-revision/
---
The typical SQL Server activation procedure is contains a <tt>WHILE (1=1)</tt> loop and exit conditions based on checking <tt>@@ROWCOUNT</tt>. Error handling is done via a <tt>BEGIN TRY ... BEGIN CATCH</tt> block. This pattern is present in many Service Broker articles on the web, including this web site, in books and in Microsoft samples:


<code language="SQL">
&lt;pre>
create procedure usp_queueHandler
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
		from [queueName];
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