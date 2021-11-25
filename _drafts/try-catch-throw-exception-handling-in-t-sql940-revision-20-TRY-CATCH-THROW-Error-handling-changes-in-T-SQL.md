---
id: 975
title: 'TRY CATCH THROW: Error handling changes in T-SQL'
date: 2010-11-23T10:16:16+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/23/940-revision-20/
permalink: /2010/11/23/940-revision-20/
---
When SQL Server 2005 introduced <tt>BEGIN TRY</tt> and <tt>BEGIN CATCH</tt> syntax, it was a huge improvement over the previous error handling based on @@ERROR check _after each statement_. Finally, T-SQL joined the rank of _programming_ languages, no more just a data access language. Experience has shown that exception handling leads to better code compared to error checks. Yes, SEH is slower, but is basically impossible to maintain the code discipline to check @@ERROR after every operation, so exception handling is just so much easier to get right. And besides, @@ERROR never had such a masterpiece article to guide you trough like <a href="http://www.microsoft.com/msj/0197/exception/exception.aspx" target="_blank">A Crash Course on the Depths of Win32™ Structured Exception Handling</a>.

But when trying to use the new TRY/CATCH exception handling in T-SQL code, one problem quickly became apparent: the CATCH block was masking the original error metadata: error number/severity/state, error text, origin line and so on. Within a CATCH block the code was only allowed to raise a \*new\* error. Sure, the original error information could be passed on in the raised error message, but only as a _message_. The all important error code was changed. This may seem like a minor issue, but turns out to have a quite serious cascading effect: the caller now has to understand the _new_ error codes raised by your code, instead of the original system error codes. If the application code was prepared to handle deadlocks (error code 1205) in a certain way (eg. retry the transaction), with a T-SQL TRY/CATCH block the deadlock error code would all of the sudden translate into something above 50000.

<!--more-->

With SQL Server 11, this is not the case anymore. <a href="http://msdn.microsoft.com/en-us/library/ee677615%28v=SQL.110%29.aspx" target="_blank"><tt>THROW</tt></a> was introduced in the language to allow the exception handling to re-throw the _original_ error information. Revisiting the stored procedure template I recommended to use for proper handling of nested transactions in the presence of exception in [Exception handling and nested transactions](http://rusanu.com/2009/06/11/exception-handling-and-nested-transactions/), here is how the template would be modified for SQL Server 11 to take advantage of <tt>THROW<tt>:</p> 

<pre>
<code class="prettyprint lang-sql">
create procedure [usp_my_procedure_name]
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
		select @error = ERROR_NUMBER()
                                   , @message = ERROR_MESSAGE()
                                   , @xstate = XACT_STATE();
		if @xstate = -1
			rollback;
		if @xstate = 1 and @trancount = 0
			rollback
		if @xstate = 1 and @trancount &gt; 0
			rollback transaction usp_my_procedure_name;

		throw;
	end catch	
end
</code>
</pre>

<h2>
  RAISERROR is deprecated
</h2>

<p>
  <i>11/23: I leave this text here as I originally wrote it, but read bellow why almost everything I say here is wrong.</i>
</p>

<p>
  With the introduction of THROW, RAISERROR was declared obsolete and put on the future deprecation list. THROW can be used instead of RAISERROR to throw a new error:
</p>

<pre>
<code class="prettyprint lang-sql">
THROW 51000, 'The record does not exist.', 1;
</code>
</pre>

<p>
  New exceptions raised with THROW will all have a severity level 16. Needless to say, exception re-thrown from a CATCH block preserve the original severity. THROW without additional arguments can only be used inside a CATCH block. THROW with explicit error number can be used in any place in code.
</p>

<p>
  But <tt>RAISERROR</tt> had a very handy feature: it could format the error message and replace, <tt>printf</tt> style, arguments into it. And since severity 0 was basically a <tt>PRINT</tt>, it was a very handy replacement for the cumbersome and archaic <tt>PRINT</tt> restriction (remember, PRINT can only print one and only one variable/message per line). But THROW does not allow for argument replacement in the message. Instead, the guidance is to use the <a href="http://msdn.microsoft.com/en-us/library/ms186788%28v=SQL.110%29.aspx" target="_blank"><tt>FORMATMESSAGE</tt></a> infrastructure. Is true that <tt>FORMATMESSAGE</tt> has localization support, but that will hardly sugar coat the sorrow pill of taking away message formatting like RAISERROR had:
</p>

<ul>
  <li>
    Application developers have to deal with localization in the application front end anyway, so they much rather deal with one uniform localization infrastructure, and that one infrastructure <i>must</i> be the one that support the front end features: resource DLLs.
  </li>
  <li>
    Database errors do not make it to the localized front end. Displaying errors about allocation failures due to file growth restrictions or page checksum validation errors are hardly of any value to the end user, and are very often disclosing information that was not supposed to leak to the front-end. Most applications make use of the database errors solely for <b>logging</b>, which is not localized in the end-user language but instead must be understood by the developers.
  </li>
  <li>
    Lacking support for constants in T-SQL makes development of code that uses <i>magic numbers</i> problematic. <tt>FORMATMESSAGE (52113, ...)</tt> what the heck is 52133? I rather have <tt>FORMATMESSAGE(ERROR_RECORD_MISSING,...)</tt>...
  </li>
  <li>
    Message IDs have no namespace. As global values in the database, the danger of conflicts between side-by-side deployed applications is always present.
  </li>
  <li>
    Message IDs have to be provisioned at application deployment time. With the deployment/setup/upgrade story for T-SQL being already in a pretty bad shape, no sane developer would add another dependency on that.
  </li>
</ul>

<p>
  Given these points, is no wonder that message ID based errors are basically unheard of in the T-SQL backed application development. I feel that the <tt>FORMATMESSAGE</tt> story as a replacement for deprecation of the <tt>RAISERROR</tt> formatting capabilities is a step backward for the new THROW syntax.
</p>

<h2>
  Update 11/23
</h2>

<p>
  As Aaron pointed out, the MSDN quote about RAISERROR is a documentation error. The function is not deprecated. Furthermore the FORMATMESSAGE function was actually enhanced to support ad-hoc formatting:
</p>

<pre>
<code class="prettyprint lang-sql">
SELECT FORMATMESSAGE('Hello, %s!', 'World');
</code>
</pre>

<p>
  Between these two additional pieces of information, my <del datetime="2010-11-23T18:07:26+00:00">rant</del> concern about the deprecation of RAISERROR and the lack of ad-hoc formatting in FORMATMESSAGE is obsolete and... deprecated.
</p>