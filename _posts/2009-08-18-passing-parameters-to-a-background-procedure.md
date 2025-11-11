---
id: 532
title: Passing Parameters to a Background Procedure
date: 2009-08-18T11:21:47+00:00
author: remus
layout: post
guid: /?p=532
permalink: /2009/08/18/passing-parameters-to-a-background-procedure/
categories:
  - Samples
  - Tutorials
tags:
  - activation
  - asynchronous procedure
  - background
  - long running
  - parameters
  - sql server
  - tsql
---
<p class="callout float-right">
  Code on GitHub: <a href="https://github.com/rusanu/async_tsql">rusanu/async_tsql</a>
</p>

I have posted previously an example [how to invoke a procedure asynchronously](/2009/08/05/asynchronous-procedure-execution) using service Broker activation. Several readers have inquired how to extend this mechanism to add parameters to the background launched procedure.

Passing parameters to a single well know procedure is easy: the parameters are be added to the message body and the activated procedure looks them up in the received XML, passing them to the called procedure. But is significantly more complex to create a generic mechanism that can pass parameters to any procedure. The problem is the type system, because the parameters have unknown types and the activated procedure has to pass proper typed parameters to the invoked procedure.

A generic solution should accept a variety of parameter types and should deal with the peculiarities of Transact-SQL parameters passing, namely the named parameters capabilities. Also the invocation wrapper <tt>usp_AsyncExecInvoke</tt> should directly accept the parameters for the desired background procedure. After considering several alternatives, I settled on the following approach:

<!--more-->

  * Rely on the generic <a href="http://msdn.microsoft.com/en-us/library/ms173829.aspx" target="_blank">sql_variant</a> data type available in SQL Server. The invocation wrapper <tt>usp_AsyncExecInvoke</tt> accept all the parameters as <tt>sql_variant</tt>.
  * Pass the background procedure parameter names explicitly to the invocation wrapper <tt>usp_AsyncExecInvoke</tt>.
  * Always used named parameters in the background procedure invocation.
  * Build a dynamic SQL batch to invoke the background procedure that deals with the required parameter type conversions.

## Accepting Parameters

The invocation wrapper <tt>usp_AsyncExecInvoke<tt> has changed the signature to accept a variable number of parameters:</p> 

<pre><code class="prettyprint lang-sql">
create procedure [usp_AsyncExecInvoke]
    @procedureName sysname
    , @p1 sql_variant = NULL, @n1 sysname = NULL
    , @p2 sql_variant = NULL, @n2 sysname = NULL
    , @p3 sql_variant = NULL, @n3 sysname = NULL
    , @p4 sql_variant = NULL, @n4 sysname = NULL
    , @p5 sql_variant = NULL, @n5 sysname = NULL
    , @token uniqueidentifier output
</code></pre>

<p>
  The new parameters are used to collect the desired parameter values <i>and names</i> to be passed to the background procedure. So sonsider you would like to make the following traditional, synchronous, call:
</p>

<pre><code class="prettyprint lang-sql">
exec usp_withParam @id = 1.0
	, @name = N'Foo'
	, @bytes = 0xBAADF00D;
</code></pre>

<p>
  The equivalent asynchronous invocation would need to pass the parameter values (1.0, 'Foo' and 0xbaadf00d) as @p1, @p2 and @p3 respectively and the parameter names (@id, @name and @bytes) as @n1, @n2 and @n3: 
  
  <pre><code class="prettyprint lang-sql">
exec usp_AsyncExecInvoke @procedureName = N'usp_withParam'
 , @p1 = 1.0, @n1 = N'@id'
 , @p2 = N'Foo', @n2='@name'
 , @p3 = 0xBAADF00D, @n3 = '@bytes'
 , @token = @token output;
</code></pre>
  
  <p>
    The invocation wrapper will pack the parameter names, values and type information into the message body. The parameter type information is deduced at run time from the actual parameters, by using the <a href="http://msdn.microsoft.com/en-us/library/ms178550.aspx" target="_blank">SQL_VARIANT_PROPERTY</a> system function. This is the XML message body corresponding to the invocation above (note how the varbinary parameter was encoded using base64 encoding):
  </p>
  
  <pre><code class="prettyprint lang-sql">
&lt;procedure>
  &lt;name>usp_withParam&lt;/name>
  &lt;parameters>
    &lt;parameter Name="@id" BaseType="numeric" Precision="2" Scale="1" MaxLength="5">1.0&lt;/parameter>
    &lt;parameter Name="@name" BaseType="nvarchar" Precision="0" Scale="0" MaxLength="6">Foo&lt;/parameter>
    &lt;parameter Name="@bytes" BaseType="varbinary" 
          Precision="0" Scale="0" MaxLength="4">uq3wDQ==&lt;/parameter>
  &lt;/parameters>
&lt;/procedure>
</code></pre>
  
  <h2>
    Extracting the Parameters
  </h2>
  
  <p>
    The activated procedure job is now significantly more complex because of the task of extracting the parameter names, types and value and constructing a proper Transact-SQL statement to invoke the original procedure. A new helper procedure was added, <tt>usp_procedureInvokeHelper</tt>, that constructs a dynamic SQL batch from a invokation message body. Here is the dynamic SQL that results from our sample invocation:
  </p>
  
  <pre><code class="prettyprint lang-sql">
declare @pt1 numeric(2,1)
declare @pt2 nvarchar(6)
declare @pt3 varbinary(4)
select @pt1=@x.value(N'(//procedure/parameters/parameter)[1]', N'numeric(2,1)');
select @pt2=@x.value(N'(//procedure/parameters/parameter)[2]', N'nvarchar(6)');
select @pt3=@x.value(N'(//procedure/parameters/parameter)[3]', N'varbinary(4)');
exec [usp_withParam]  @id=@pt1,@name=@pt2,@bytes=@pt3
</code></pre>
  
  <h2>
    Limitations
  </h2>
  
  <p>
    Because the invocation wrapper uses sql_variant typed parameters, all sql_variant restrictions apply to this sample. Most notably the wrapper will not accept XML type parameters nor large data types like varchar(max) or varbinary(max). It is possible to work around these limitations but it would complicate my sample beyond the point of being usefull as an example how to pass the parameters.
  </p>
  
  <p>
    Because the error handling of the activated procedure will rollback in case of error, passing in a message that results in a procedure invocation error will cause the activation to rollback repeatedly and the poison message detection mechanism will kick in, deactivating the queue. A trivial example is if we omit the required @id parameter for our sample invocation.
  </p>
  
  <h2>
    The code
  </h2>
  
  <pre><code class="prettyprint lang-sql">
create table [AsyncExecResults] (
 [token] uniqueidentifier primary key
 , [submit_time] datetime not null
 , [start_time] datetime null
 , [finish_time] datetime null
 , [error_number] int null
 , [error_message] nvarchar(2048) null);
go

create queue [AsyncExecQueue];
go

create service [AsyncExecService] on queue [AsyncExecQueue] ([DEFAULT]);
GO

-- Dynamic SQL helper procedure
-- Extracts the parameters from the message body
-- Creates the invocation Transact-SQL batch
-- Invokes the dynmic SQL batch
create procedure [usp_procedureInvokeHelper] (@x xml)
as
begin
    set nocount on;

    declare @stmt nvarchar(max)
        , @stmtDeclarations nvarchar(max)
        , @stmtValues nvarchar(max)
        , @i int
        , @countParams int
        , @namedParams nvarchar(max)
        , @paramName sysname
        , @paramType sysname
        , @paramPrecision int
        , @paramScale int
        , @paramLength int
        , @paramTypeFull nvarchar(300)
        , @comma nchar(1)

    select @i = 0
        , @stmtDeclarations = N''
        , @stmtValues = N''
        , @namedParams = N''
        , @comma = N''

    declare crsParam cursor forward_only static read_only for
        select x.value(N'@Name', N'sysname')
            , x.value(N'@BaseType', N'sysname')
            , x.value(N'@Precision', N'int')
            , x.value(N'@Scale', N'int')
            , x.value(N'@MaxLength', N'int')
        from @x.nodes(N'//procedure/parameters/parameter') t(x);
    open crsParam;

    fetch next from crsParam into @paramName
        , @paramType
        , @paramPrecision
        , @paramScale
        , @paramLength;
    while (@@fetch_status = 0)
    begin
        select @i = @i + 1;

        select @paramTypeFull = @paramType +
            case
            when @paramType in (N'varchar'
                , N'nvarchar'
                , N'varbinary'
                , N'char'
                , N'nchar'
                , N'binary') then
                N'(' + cast(@paramLength as nvarchar(5)) + N')'
            when @paramType in (N'numeric') then
                N'(' + cast(@paramPrecision as nvarchar(10)) + N',' +
                cast(@paramScale as nvarchar(10))+ N')'
            else N''
            end;

        -- Some basic sanity check on the input XML
        if (@paramName is NULL
            or @paramType is NULL
            or @paramTypeFull is NULL
            or charindex(N'''', @paramName) > 0
            or charindex(N'''', @paramTypeFull) > 0)
            raiserror(N'Incorrect parameter attributes %i: %s:%s %i:%i:%i'
                , 16, 10, @i, @paramName, @paramType
                , @paramPrecision, @paramScale, @paramLength);

        select @stmtDeclarations = @stmtDeclarations + N'
declare @pt' + cast(@i as varchar(3)) + N' ' + @paramTypeFull
            , @stmtValues = @stmtValues + N'
select @pt' + cast(@i as varchar(3)) + N'=@x.value(
    N''(//procedure/parameters/parameter)[' + cast(@i as varchar(3)) 
                + N']'', N''' + @paramTypeFull + ''');'
            , @namedParams = @namedParams + @comma + @paramName
                + N'=@pt' + cast(@i as varchar(3));

        select @comma = N',';

        fetch next from crsParam into @paramName
            , @paramType
            , @paramPrecision
            , @paramScale
            , @paramLength;
    end

    close crsParam;
    deallocate crsParam;        

    select @stmt = @stmtDeclarations + @stmtValues + N'
exec ' + quotename(@x.value(N'(//procedure/name)[1]', N'sysname'));

    if (@namedParams != N'')
        select @stmt = @stmt + N' ' + @namedParams;

    exec sp_executesql @stmt, N'@x xml', @x;
end
go

create procedure usp_AsyncExecActivated
as
begin
    set nocount on;
    declare @h uniqueidentifier
        , @messageTypeName sysname
        , @messageBody varbinary(max)
        , @xmlBody xml
        , @startTime datetime
        , @finishTime datetime
        , @execErrorNumber int
        , @execErrorMessage nvarchar(2048)
        , @xactState smallint
        , @token uniqueidentifier;

    begin transaction;
    begin try;
        receive top(1)
            @h = [conversation_handle]
            , @messageTypeName = [message_type_name]
            , @messageBody = [message_body]
            from [AsyncExecQueue];
        if (@h is not null)
        begin
            if (@messageTypeName = N'DEFAULT')
            begin
                -- The DEFAULT message type is a procedure invocation.
                --
                select @xmlBody = CAST(@messageBody as xml);

                save transaction usp_AsyncExec_procedure;
                select @startTime = GETUTCDATE();
                begin try
                    exec [usp_procedureInvokeHelper] @xmlBody;
                end try
                begin catch
                -- This catch block tries to deal with failures of the procedure execution
                -- If possible it rolls back to the savepoint created earlier, allowing
                -- the activated procedure to continue. If the executed procedure
                -- raises an error with severity 16 or higher, it will doom the transaction
                -- and thus rollback the RECEIVE. Such case will be a poison message,
                -- resulting in the queue disabling.
                --
                select @execErrorNumber = ERROR_NUMBER(),
                    @execErrorMessage = ERROR_MESSAGE(),
                    @xactState = XACT_STATE();
                if (@xactState = -1)
                begin
                    rollback;
                    raiserror(N'Unrecoverable error in procedure: %i: %s', 16, 10,
                        @execErrorNumber, @execErrorMessage);
                end
                else if (@xactState = 1)
                begin
                    rollback transaction usp_AsyncExec_procedure;
                end
                end catch

                select @finishTime = GETUTCDATE();
                select @token = [conversation_id]
                    from sys.conversation_endpoints
                    where [conversation_handle] = @h;
                if (@token is null)
                begin
                    raiserror(N'Internal consistency error: conversation not found', 16, 20);
                end
                update [AsyncExecResults] set
                    [start_time] = @starttime
                    , [finish_time] = @finishTime
                    , [error_number] = @execErrorNumber
                    , [error_message] = @execErrorMessage
                    where [token] = @token;
                if (0 = @@ROWCOUNT)
                begin
                    raiserror(N'Internal consistency error: token not found', 16, 30);
                end
                end conversation @h;
            end
            else if (@messageTypeName = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog')
            begin
                end conversation @h;
            end
            else if (@messageTypeName = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error')
            begin
                declare @errorNumber int
                    , @errorMessage nvarchar(4000);
                select @xmlBody = CAST(@messageBody as xml);
                with xmlnamespaces (DEFAULT N'http://schemas.microsoft.com/SQL/ServiceBroker/Error')
                select @errorNumber = @xmlBody.value ('(/Error/Code)[1]', 'INT'),
                    @errorMessage = @xmlBody.value ('(/Error/Description)[1]', 'NVARCHAR(4000)');
                -- Update the request with the received error
                select @token = [conversation_id] 
                    from sys.conversation_endpoints 
                    where [conversation_handle] = @h;
                update [AsyncExecResults] set
                    [error_number] = @errorNumber
                    , [error_message] = @errorMessage
                    where [token] = @token;
                end conversation @h;
             end
           else
           begin
                raiserror(N'Received unexpected message type: %s', 16, 50, @messageTypeName);
           end
        end
        commit;
    end try
    begin catch
        declare @error int
         , @message nvarchar(2048);
        select @error = ERROR_NUMBER()
            , @message = ERROR_MESSAGE()
            , @xactState = XACT_STATE();
        if (@xactState &lt;> 0)
        begin
         rollback;
        end;
        raiserror(N'Error: %i, %s', 1, 60,  @error, @message) with log;
    end catch
end
go

alter queue [AsyncExecQueue]
    with activation (
    procedure_name = [usp_AsyncExecActivated]
    , max_queue_readers = 1
    , execute as owner
    , status = on);
go

-- Helper function to create the XML element 
-- for a passed in parameter
create function [dbo].[fn_DescribeSqlVariant] (
 @p sql_variant
 , @n sysname)
returns xml
with schemabinding
as
begin
 return (
 select @n as [@Name]
  , sql_variant_property(@p, 'BaseType') as [@BaseType]
  , sql_variant_property(@p, 'Precision') as [@Precision]
  , sql_variant_property(@p, 'Scale') as [@Scale]
  , sql_variant_property(@p, 'MaxLength') as [@MaxLength]
  , @p
  for xml path('parameter'), type)
end
GO

-- Invocation wrapper. Accepts arbitrary
-- named parameetrs to be passed to the
-- background procedure
create procedure [usp_AsyncExecInvoke]
    @procedureName sysname
    , @p1 sql_variant = NULL, @n1 sysname = NULL
    , @p2 sql_variant = NULL, @n2 sysname = NULL
    , @p3 sql_variant = NULL, @n3 sysname = NULL
    , @p4 sql_variant = NULL, @n4 sysname = NULL
    , @p5 sql_variant = NULL, @n5 sysname = NULL
    , @token uniqueidentifier output
as
begin
    declare @h uniqueidentifier
     , @xmlBody xml
        , @trancount int;
    set nocount on;

 set @trancount = @@trancount;
    if @trancount = 0
        begin transaction
    else
        save transaction usp_AsyncExecInvoke;
    begin try
        begin dialog conversation @h
            from service [AsyncExecService]
            to service N'AsyncExecService', 'current database'
            with encryption = off;
        select @token = [conversation_id]
            from sys.conversation_endpoints
            where [conversation_handle] = @h;

        select @xmlBody = (
            select @procedureName as [name]
            , (select * from (
                select [dbo].[fn_DescribeSqlVariant] (@p1, @n1) AS [*] 
                    WHERE @p1 IS NOT NULL
                union all select [dbo].[fn_DescribeSqlVariant] (@p2, @n2) AS [*] 
                    WHERE @p2 IS NOT NULL
                union all select [dbo].[fn_DescribeSqlVariant] (@p3, @n3) AS [*] 
                    WHERE @p3 IS NOT NULL
                union all select [dbo].[fn_DescribeSqlVariant] (@p4, @n4) AS [*] 
                    WHERE @p4 IS NOT NULL
                union all select [dbo].[fn_DescribeSqlVariant] (@p5, @n5) AS [*] 
                    WHERE @p5 IS NOT NULL
                ) as p for xml path(''), type
            ) as [parameters]
            for xml path('procedure'), type);
        send on conversation @h (@xmlBody);
        insert into [AsyncExecResults]
            ([token], [submit_time])
            values
            (@token, getutcdate());
    if @trancount = 0
        commit;
    end try
    begin catch
        declare @error int
            , @message nvarchar(2048)
            , @xactState smallint;
        select @error = ERROR_NUMBER()
            , @message = ERROR_MESSAGE()
            , @xactState = XACT_STATE();
        if @xactState = -1
            rollback;
        if @xactState = 1 and @trancount = 0
            rollback
        if @xactState = 1 and @trancount > 0
            rollback transaction usp_my_procedure_name;

        raiserror(N'Error: %i, %s', 16, 1, @error, @message);
    end catch
end
go

-- Sample invocation example
-- The usp_withParam will insert 
-- all the received parameters into this table
-- 
create table [withParam] (
    id numeric(4,1) NULL
    , name varchar(150) NULL
    , date datetime  NULL
    , value int NULL
    , bytes varbinary(max) NULL);
go

create procedure usp_withParam
 @id numeric(4,1)
 , @name varchar(150)
 , @date datetime = NULL
 , @value int = 0
 , @bytes varbinary(max) = NULL
as
begin
    insert into [withParam] (
        id
        , name
        , date
        , value
        , bytes)
     select @id as [id]
      , @name as [name]
      , @date as [date]
      , @value as [value]
      , @bytes as [bytes]
end
go

declare @token uniqueidentifier;

exec usp_AsyncExecInvoke @procedureName = N'usp_withParam'
 , @p1 = 1.0, @n1 = N'@id'
 , @p2 = N'Foo', @n2='@name'
 , @p3 = 0xBAADF00D, @n3 = '@bytes'
 , @token = @token output;

waitfor delay '00:00:05';

select * from AsyncExecResults;
select * from withParam;

go
</code>
</pre>