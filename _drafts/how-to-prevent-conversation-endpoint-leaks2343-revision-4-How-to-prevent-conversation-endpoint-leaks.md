---
id: 2347
title: How to prevent conversation endpoint leaks
date: 2014-03-31T01:30:48+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/03/31/2343-revision-4/
permalink: /2014/03/31/2343-revision-4/
---
One of the most common complains about using Service Broker in production is when administrators discover, usually after some months of usage, that [<tt>sys.conversations_endpoints</tt>](http://technet.microsoft.com/en-us/library/ms176082.aspx) grows out of control with CLOSED conversations that are never cleaned up. I will show how this case occurs and what to do to fix it.

<!--more-->

# A message exchange pattern that leaks endpoints

I have a SQL Server instance (this happens to be SQL Server 2012) on which I have enabled the Service Broker endpoint. I&#8217;m going to set up a loopback conversation, one that is forced to go on the netwrok even though both initiator and target service are local:

source.sql

<pre><code class="prettyprint lang-sql">
create database [source];
go

use [source];
go

create queue [q];
go

create service [source_service] on queue [q];
go

create route [target] with service_name = N'target_service', 
	address = 'tcp://localhost:4022';
go

create procedure usp_source
as
begin
	set nocount on;
	declare @h uniqueidentifier, @mt sysname;

	begin transaction;

	receive top(1) @mt = message_type_name, @h = conversation_handle from q;
	if (@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog' or
		@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error')
	begin
		end conversation @h;
	end
	commit
end
go

alter queue q with activation (
	status = on, 
	max_queue_readers = 1, 
	procedure_name = [usp_source], 
	execute as  owner);
go
&lt;/pre>
&lt;p></code></p>


<p>
  target.sql
</p>


<pre><code class="prettyprint lang-sql">
create database [target];
go

use [target];
go

create queue [q];
go

create service [target_service] on queue [q] ([DEFAULT]);
go

grant send on service::[target_service] to [public];
go

create route [source] with service_name = N'source_service', 
	address = 'tcp://localhost:4022';
go

create procedure usp_target
as
begin
	set nocount on;
	declare @h uniqueidentifier, @mt sysname;

	begin transaction;

	receive top(1) @mt = message_type_name, @h = conversation_handle from q;
	if (@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog' or
		@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error')
	begin
		end conversation @h;
	end
	commit
end
go

alter queue q with activation (
	status = on, 
	max_queue_readers = 1, 
	procedure_name = [usp_target], 
	execute as  owner);
go
&lt;/pre>
&lt;p></code></p>


<p>
  This is pretty much as simple as it gets, is pair of services called <tt>source_service</tt> and <tt>target_service</tt> which have set up activation on their queue to simply end conversations when the <tt>EndDialog</tt> or <tt>Error</tt> messages are received (the very minimum requirement any service should handle). I'm using the <tt>DEFAULT</tt> contract. Next I'm going to send a message and immediately end the dialog:
</p>


<p>
  dialog.sql
</p>


<pre><code class="prettyprint lang-sql">
use [source];
go

begin tran;
declare @h uniqueidentifier;
begin dialog conversation @h
	from service [source_service]
	to service N'target_service'
	with encryption = off;
send on conversation @h;
end conversation @h;
commit;
go
&lt;/pre>
&lt;p></code></p>


<p>
  Lets check the target database conversation endpoints:
</p>


<p>
  check.sql
</p>


<pre><code class="prettyprint lang-sql">
select  lifetime, state_desc, security_timestamp 
	from [target].sys.conversation_endpoints;

lifetime                state_desc                                   security_timestamp
----------------------- ----------------------------------------- -----------------------
2082-04-18 11:41:08.987 CLOSED                                       1900-01-01 00:00:00.000

(1 row(s) affected)
&lt;/pre>
&lt;p></code></p>