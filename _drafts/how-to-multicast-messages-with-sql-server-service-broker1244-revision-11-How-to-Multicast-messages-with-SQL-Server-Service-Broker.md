---
id: 2040
title: How to Multicast messages with SQL Server Service Broker
date: 2011-07-20T18:40:39+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/20/1244-revision-11/
permalink: /2011/07/20/1244-revision-11/
---
Starting with SQL Server 11 the the <a href="http://msdn.microsoft.com/en-us/library/ms188407%28v=SQL.110%29.aspx"target="_blank"><tt>SEND</tt></a> verb has a new syntax and accepts multiple dialogs handles to send on:

<pre>SEND
   ON CONVERSATION [(]conversation_handle [,.. @conversation_handle_n][)]
   [ MESSAGE TYPE message_type_name ]
   [ ( message_body_expression ) ]
[ ; ]
</pre>

With this syntax enhancement you can send a message to multiple destinations. This is not different from sending the same message multiple times. From the application point of view issuing one single SEND on 10 dialog handles is exactly the same as issuing 10 SEND statements on one dialog handle at a time. The improvement is in the <tt>sys.transmission_queue</tt>: issuing SEND multiple time would create multiple copies of the message body to be sent. By contrast the one single SEND on multiple handles will only store the message body once. We can see this if we look at the definition of sys.tranmission_queue in SQL Server 11:

<pre>sp_helptext 'sys.transmission_queue'

CREATE VIEW sys.transmission_queue AS
	SELECT conversation_handle = S.handle,
		to_service_name = Q.tosvc,
		to_broker_instance = Q.tobrkrinst,
		from_service_name = Q.fromsvc,
		service_contract_name = Q.svccontr,
		enqueue_time = Q.enqtime,
		message_sequence_number = Q.msgseqnum,
		message_type_name = Q.msgtype,
		is_conversation_error = sysconv(bit, Q.status & 2),
		is_end_of_dialog = sysconv(bit, Q.status & 4),
		<span style="background:yellow">message_body = ISNULL(Q.msgbody, B.msgbody)</span>,
		transmission_status = GET_TRANSMISSION_STATUS (S.handle),
		priority = R.priority
	FROM sys.sysxmitqueue Q
	<span style="background:yellow">LEFT JOIN sys.sysxmitbody B WITH (NOLOCK) ON Q.msgref = B.msgref</span>
	INNER JOIN sys.sysdesend S WITH (NOLOCK) 
             ON Q.dlgid = S.diagid AND Q.finitiator = S.initiator
	INNER JOIN sys.sysdercv R WITH (NOLOCK) 
             ON Q.dlgid = R.diagid AND Q.finitiator = R.initiator
	WHERE is_member('db_owner') = 1
</pre>

Compare this with the same view definition in SQL Server 2008 R2:

<pre>CREATE VIEW sys.transmission_queue AS
	SELECT conversation_handle = S.handle,
		to_service_name = Q.tosvc,
		to_broker_instance = Q.tobrkrinst,
		from_service_name = Q.fromsvc,
		service_contract_name = Q.svccontr,
		enqueue_time = Q.enqtime,
		message_sequence_number = Q.msgseqnum,
		message_type_name = Q.msgtype,
		is_conversation_error = sysconv(bit, Q.status & 2),
		is_end_of_dialog = sysconv(bit, Q.status & 4),
		<span style="background:yellow">message_body = Q.msgbody</span>,
		transmission_status = GET_TRANSMISSION_STATUS (S.handle),
		priority = R.priority
	FROM sys.sysxmitqueue Q
	INNER JOIN sys.sysdesend S WITH (NOLOCK) 
               ON Q.dlgid = S.diagid AND Q.finitiator = S.initiator
	INNER JOIN sys.sysdercv R WITH (NOLOCK) 
               ON Q.dlgid = R.diagid AND Q.finitiator = R.initiator
	WHERE is_member('db_owner') = 1
</pre>

You can see how in SQL Server 11 the message body was separated into a new system table (<tt>sys.sysxmitbody</tt>). Multicast SEND will create multiple entries in <tt>sys.sysxmitqueue</tt> (one for each dialog on which the message was multicasted) but only one entry in <tt>sys.sysxmitbody</tt>. Such a normalized storage scheme saves space consumed and, more importantly, amount of log generated during the SEND.

## The Reversed Dialog pattern in publish-subscribe

The typical dialog pattern in pub-sub systems is for the subscriber to start the dialog and send an initial &#8216;subscribe&#8217; message, then the subscription content is being delivered from the target (the publisher) to the initiator (the subscriber). I call this the Reverse Dialog Pattern because messages flow from the target to the initiator. Lets show with an example. We&#8217;ll create a publisher service that broadcast some important content to which services can subscribe to receive it. To spice it up, we&#8217;ll use a tag system to subscribe to optional content: all content is distributed with a list of associated tags, all subscribers specify the tag they&#8217;re interested in. Tag matching is done using the <a href="http://msdn.microsoft.com/en-us/library/ms179859.aspx" target="_blank">LIKE</a> syntax, so that subscribers can specify <tt>'%'</tt> as a mean to subscribe to all content.

### The Publisher Service

<pre>create message type subscription_request validation = none;
create message type subscription_content validation = well_formed_xml;

create contract distribution
	(subscription_request sent by initiator,
	subscription_content sent by target);

create queue publisher;	
create service publisher on queue publisher (distribution);
go

create table subscriptions (
	subscription_id int not null identity(1,1),
	tag nvarchar(50) not null,
	conversation_handle uniqueidentifier not null,
	constraint pk_subscriptions primary key (subscription_id),
	constraint unq_conversation_handle unique (conversation_handle, tag));
go

create procedure usp_publisher_handler
as
begin
	declare @mt sysname, @dh uniqueidentifier, @mb varbinary(max);
	begin try
		begin transaction;
		receive top(1) 
			@mt = message_type_name,
			@dh = conversation_handle,
			@mb = message_body
			from publisher;
		if (@mt = N'subscription_request')
		begin
			insert into subscriptions (conversation_handle, tag) 
                                   values (@dh, cast(@mb as nvarchar(50)));
		end
		else if(@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
			or @mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog')
		begin
			delete from subscriptions 
				where conversation_handle  = @dh;
			end conversation @dh;
		end
		commit
	end try
	begin catch
		declare @xact_state int = xact_state();
		if @xact_state &lt;> 0
		begin
			rollback;
		end
	end catch
end
go

alter queue publisher with activation (
	status = on,
	max_queue_readers = 1,
	procedure_name = usp_publisher_handler, 
	execute as owner);
go
</pre>

The publisher service is straight forward: it uses the <tt>subscriptions</tt> table to keep track of subscribers. The activated procedure associated with the publisher service processes the subscription_request messages and adds the request dialog to the subscriptions table. The request message body is the tag the subscriber is interested in.

### The Publish Content procedure

<pre>create type publish_tags_type as table (
	tag nvarchar(50) not null primary key);
go

create procedure usp_publish_content
	@content xml,
	@tags publish_tags_type readonly
as	
begin
	declare @sql nvarchar(max) = N'send on conversation (';
	declare @cnt int = 0;
	declare @dh uniqueidentifier;
	declare @comma nvarchar(2) = N'';
	
	declare crs cursor static read_only forward_only for
		select distinct conversation_handle
		from subscriptions s
		join @tags t on t.tag like s.tag;
		
	open crs;
	fetch next from crs into @dh;
	while 0 = @@fetch_status
	begin
	
		set @sql += @comma  + N'''' + cast(@dh as nvarchar(36)) + N'''';
		set @comma = N', ';
		set @cnt += 1;
		fetch next from crs into @dh;
	end
	close crs;
	deallocate crs;
	
	if @cnt > 0
	begin
		set @sql+= N') message type subscription_content (@content)';
		exec sp_executesql @sql, N'@content xml', @content;
	end
end
go
</pre>

The publish content procedure takes a content to be distributed and the list of tags under which the content is distributed and sends the content to all interested subscribers. One single multicast SEND is used to reach all subscribers. Dynamic SQL is used to build the multicast SEND statement.

### Adding subscribers

<pre>declare @i int = 0;
declare @sql nvarchar(max);
while @i &lt; 10
begin
	set @sql = N'create queue subscriber_' + cast(@i as nvarchar(20)) + N';
		create service subscriber_' + cast(@i as nvarchar(20)) + N' 
                            on queue subscriber_'+cast(@i as nvarchar(20)) + N';';
	exec sp_executesql @sql;
	set @sql = N'declare @dh uniqueidentifier;
		begin dialog @dh 
			from service subscriber_' + cast(@i as nvarchar(20)) + N'
			to service N''publisher''
			on contract distribution
			with encryption = off;
		send on conversation @dh message type subscription_request 
                    (''' +case @i%5 when 0 then N'%' else nchar(@i + 65) end + ''');';
	exec sp_executesql @sql;
	set @i += 1;
end
go
</pre>

This snipped adds 10 subscribers interested in tags 'B', 'C', 'D' etc. The first and fifth subscribers are interested in everything ('%'). We can see the subscribers were added to the subscriptions table by the publisher activated procedure:

<pre>select * from subscriptions

subscription_id tag                               conversation_handle
--------------- ----------------- -------------------------------------
1               %                          AFC62EF2-35B3-E011-8EED-001C25160E57
2               B                          B3C62EF2-35B3-E011-8EED-001C25160E57
3               C                          B7C62EF2-35B3-E011-8EED-001C25160E57
4               D                          BBC62EF2-35B3-E011-8EED-001C25160E57
5               E                          BFC62EF2-35B3-E011-8EED-001C25160E57
6               %                          C3C62EF2-35B3-E011-8EED-001C25160E57
7               G                          C7C62EF2-35B3-E011-8EED-001C25160E57
8               H                          CBC62EF2-35B3-E011-8EED-001C25160E57
9               I                          CFC62EF2-35B3-E011-8EED-001C25160E57
10              J                          D3C62EF2-35B3-E011-8EED-001C25160E57
</pre>

### A test multicast message

<pre>declare @tags publish_tags_type;
insert into @tags (tag) values ('A'), ('B'), ('C');
exec usp_publish_content N'&lt;Hello/>', @tags;
go
</pre>

With this one call we notified all subscribers interested, with one single multicast SEND. We can check which of the subscribers got the content:

<pre>declare @i int = 0;
declare @sql nvarchar(max) = N'', @union nvarchar(20) = N'';
while @i &lt; 10
begin
	set @sql += @union + N'select 
              ''subscriber_' + cast(@i as nvarchar(20)) + N''' as subscriber, 
               count(*) as count 
               from subscriber_' + cast(@i as nvarchar(20));
	set @union = ' union all ';
	set @i += 1;
end
exec sp_executesql @sql;
go

subscriber   count
------------ -----------
subscriber_0 1
subscriber_1 1
subscriber_2 1
subscriber_3 0
subscriber_4 0
subscriber_5 1
subscriber_6 0
subscriber_7 0
subscriber_8 0
subscriber_9 0

(10 row(s) affected)
</pre>

We can see that subscriber\_2 and subscriber\_3 each got a message since the tags they're interested are 'B' and 'C' which both match a tag set by the publisher. Subscribers 1 and 5 eahc got a message because they're interested in _any_ tag.

This pattern of publish-subscribe is not new and similar applications could be built with SQL Server Service Broker in SQL Server 2005, 2008 and 2008R2. But with SQL Server 11 the distribution is more efficient and can scale and perform better as the message bodies are not inserted and deleted multiple times, once for each subscriber, in the publisher's transmission queue.