---
id: 1249
title: How to Multicast messages with SQL Server Service Broker
date: 2011-07-20T12:58:37+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/20/1244-revision-5/
permalink: /2011/07/20/1244-revision-5/
---
Starting with SQL Server 11 the the <a href="http://msdn.microsoft.com/en-us/library/ms188407%28v=SQL.110%29.aspx"target="_blank"><tt>SEND</tt></a> verb has a new syntax and accepts multiple dialogs handles to send on:

<pre>SEND
   ON CONVERSATION [(]conversation_handle [,.. @conversation_handle_n][)]
   [ MESSAGE TYPE message_type_name ]
   [ ( message_body_expression ) ]
[ ; ]
</pre>

With this syntax enhancement you can send a message to multiple destinations. This is not different from sending the same message multiple times. From the application point of view issuing one single SEND on 10 dialog handles is exactly the same as issuing 10 SEND statements on one dialog handle at a time. the improvement is in the <tt>sys.transmission_queue</tt>: issuing SEND multiple time would create multiple copies of the message body to be sent. By contrast the one single SEND on multiple handles will only store the message body once. We can see this if we look at the definition of sys.tranmission_queue in SQL Server 11:

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
	INNER JOIN sys.sysdesend S WITH (NOLOCK) on Q.dlgid = S.diagid AND Q.finitiator = S.initiator
	INNER JOIN sys.sysdercv R WITH (NOLOCK) ON Q.dlgid = R.diagid AND Q.finitiator = R.initiator
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
	INNER JOIN sys.sysdesend S WITH (NOLOCK) on Q.dlgid = S.diagid AND Q.finitiator = S.initiator
	INNER JOIN sys.sysdercv R WITH (NOLOCK) ON Q.dlgid = R.diagid AND Q.finitiator = R.initiator
	WHERE is_member('db_owner') = 1
</pre>

You can see how in SQL Server 11 the message body was separated into a new system table (<tt>sys.sysxmitbody</tt>). Multicast SEND will create one entry in <tt>sys.sysxmitqueue</tt> for each dialog on which the message was multicasted, but only one entry in <tt>sys.sysxmitbody </tt>. Such a normalized storage scheme saves space consumed and, more importantly, amount of log generated during the SEND.

## The Reversed Dialog pattern in publish-subscribe dialogs</p>