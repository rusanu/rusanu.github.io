---
id: 2344
title: How to prevent conversation endpoint leaks
date: 2014-03-31T01:25:10+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/03/31/2343-revision/
permalink: /2014/03/31/2343-revision/
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
  This is pretty much as simple as it gets, is pair of services called <tt>source_service</tt> and <tt>target_service</tt> which have set up activation on their queue to simply end conversations when the <tt>EndDialog</tt> or <tt>Error</</p>