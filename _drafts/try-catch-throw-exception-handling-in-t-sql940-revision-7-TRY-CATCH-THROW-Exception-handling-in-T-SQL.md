---
id: 953
title: 'TRY-CATCH-THROW: Exception handling in T-SQL'
date: 2010-11-22T07:56:37+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/22/940-revision-7/
permalink: /2010/11/22/940-revision-7/
---
When SQL Server 2005 introduced <tt>BEGIN TRY</tt> and <tt>BEGIN CATCH</tt> syntax, it was a huge improvement over the previous error handling based on @@ERROR check _after each statement_. Finally, T-SQL joined the rank of _programming_ languages, no more just a data access language. Experience has shown that exception handling leads to better code compared to error checks. Yes, SEH is slower, but is basically impossible to maintain the code discipline to check @@ERROR after every operation, so exception handling is just so much easier to get right. And besides, @@ERROR never had such a masterpiece article to guide you trough like <a href="http://www.microsoft.com/msj/0197/exception/exception.aspx" target="_blank">A Crash Course on the Depths of Win32™ Structured Exception Handling</a>.

But when trying to use the new TRY/CATCH exception handling in T-SQL code, one problem quickly became apparent: the CATCH block was masking the original error metadata: error number/severity/state, error text, origin line and so on. Within a CATCH block the code was only allowed to raise a \*new\* error. Sure, the original error information could be passed on in the raised error message, but only as a _message_. The all important error code was changed. This may seem like a minor issue, but turns out to have a quite serious cascading effect: the caller now has to understand the _new_ error codes raised by your code, instead of the original system error codes. If the application code was prepared to handle deadlocks (error code 1205) in a certain way (eg. retry the transaction), with a T-SQL TRY/CATCH block the deadlock error code would all of the sudden translate into something above 50000.

With SQL Server 11, this is not the case anymore. <a href="http://msdn.microsoft.com/en-us/library/ee677615%28v=SQL.110%29.aspx" target="_blank"><tt>THROW</tt>/</a> was introduced in the language to allow the exception handling to re-throw the _original_ error information. Revisiting the stored procedure template I recommended to use for proper handling of nested transactions in the presence of exception in [Exception handling and nested transactions](http://rusanu.com/2009/06/11/exception-handling-and-nested-transactions/), here is how the template would be modified for SQL Server 11 to take advantage of <tt>THROW<tt>:</p> 

<pre>create procedure [usp_my_procedure_name]
as
begin
	set nocount on;
	declare @trancount int;
	set @trancount = @@trancount;
	begin try
		if @trancount = 0
			begin transaction
		else
			save transaction usp_my_procedure_name;

		-- Do the actual work here
	
lbexit:
		if @trancount = 0	
			commit;
	end try
	begin catch
		declare @error int, @message varchar(4000), @xstate int;
		select @error = ERROR_NUMBER(), @message = ERROR_MESSAGE(), @xstate = XACT_STATE();
		if @xstate = -1
			rollback;
		if @xstate = 1 and @trancount = 0
			rollback
		if @xstate = 1 and @trancount &gt; 0
			rollback transaction usp_my_procedure_name;

		throw;
	end catch	
end
</pre>