---
id: 2358
title: How to prevent conversation endpoint leaks
date: 2014-03-31T03:39:07+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/03/31/2343-revision-15/
permalink: /2014/03/31/2343-revision-15/
---
One of the most common complains about using Service Broker in production is when administrators discover, usually after some months of usage, that [<tt>sys.conversations_endpoints</tt>](http://technet.microsoft.com/en-us/library/ms176082.aspx) grows out of control with CLOSED conversations that are never cleaned up. I will show how this case occurs and what to do to fix it.

<!--more-->

# A message exchange pattern that leaks endpoints

I have a SQL Server instance (this happens to be SQL Server 2012) on which I have enabled the Service Broker endpoint. I&#8217;m going to set up a loopback conversation, one that is forced to go on the netwrok even though both initiator and target service are local:

[source.sql](https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Leak/source.sql)

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
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Leak/target.sql">target.sql</a>
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
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Leak/dialog.sql">dialog.sql</a>
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
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/check.sql">check.sql</a>
</p>


<pre><code class="prettyprint lang-sql">
select  lifetime, state_desc, security_timestamp 
	from [target].sys.conversation_endpoints;

lifetime                state_desc                                   security_timestamp
----------------------- ----------------------------------------- -----------------------
2082-04-18 11:41:08.987 CLOSED                                    1900-01-01 00:00:00.000

(1 row(s) affected)
&lt;/pre>
&lt;p></code></p>


<p class="callout float-right">
  Fire and Forget will leak CLOSED conversations
</p>


<p>
  The target conversation endpoint is in CLOSED state, but notice that the <tt>security_timestamp</tt> field is unitialized . The <tt>security_timestamp</tt> exists to prevent dialog replays (either as an malicious attack or as a configuration mistake) which would cause the dialog 'resurrect' if the first message is retried (or 'replayed'). The target conversation endpoint cannot be deleted before the datetime in the security timestamp field, which makes it safe in case of retry/replay. This field is initialized when the message from the initiator contains a special flag set that instructs the target that the initiator has seen the acknowledgement of the first message and had deleted the message 0 from its transmission queue, and thus will not re-send it. When the target receives a message with this flag set, it initializes the security timestamp with current time plus 30 minutes. You may have noticed that the message exchange pattern in my example is the dreaded <a href="http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/">fire-and-forget pattern</a>. In this pattern the initiator never has a chance to send a message with the above mentioned flag set and thus the target never has a chance to initiate the conversation security timestamp. This endpoint will be reclaimed on April 18th 2082, because that is the conversation lifetime. In case you wonder that date comes from adding MAX_INT32 (ie. 2147483647) seconds to the current date.
</p>


<p>
  From an operational point of view this target endpoint is 'leaked'. It will consume DB space and the system will refrain from deleting it for quite some time. Repeat this visious exchange pattern several thousand times per hour and in a few days your target database will simply run out of space. After frantic investigation you discover the culprit is <tt>sys.conversation_endpoints</tt> and you send me an email asking for solutions. Happens about once every week...
</p>


<h1>
  Solution 1: specify a lifetime
</h1>


<p>
  Using the very same setup, I'll add a trivial change: I will specify a 60 seconds lifetime for the conversation:
</p>


<pre><code class="prettyprint lang-sql">
use [source];
go

begin tran;
declare @h uniqueidentifier;
begin dialog conversation @h
	from service [source_service]
	to service N'target_service'
	with encryption = off, lifetime = 60;
send on conversation @h;
end conversation @h;
commit;
go
&lt;/pre>
&lt;p></code></p>


<p>
  After I let the conversation messages to be exchanged, I'm checking again the endpoints:
</p>


<pre><code class="prettyprint lang-sql">
select  lifetime, state_desc, security_timestamp 
	from [target].sys.conversation_endpoints;

lifetime                state_desc                                           security_timestamp
----------------------- ---------------------------------------------------- -----------------------
2082-04-18 11:41:08.987 CLOSED                                               1900-01-01 00:00:00.000
2014-03-31 08:47:37.947 CLOSED                                               1900-01-01 00:00:00.000

(2 row(s) affected)
&lt;/pre>
&lt;p></code></p>


<p>
  Notice how the second conversation endpoint has a lifetime that is in the near future (one minute). Even though the security timestamp is uninitialized, the conversation endpoint <b>will be reclaimed in 30 minutes after the lifetime expires</b>. I just had lunch (chicken noodle soup, grill salmon and wild rice, apple tart), and when I check again, the endpoint is gone:
</p>


<pre><code class="prettyprint lang-sql">
select  lifetime, state_desc, security_timestamp 
	from [target].sys.conversation_endpoints;

select getutcdate()

lifetime                state_desc                                         security_timestamp
----------------------- -------------------------------------------------- -----------------------
2082-04-18 11:41:08.987 CLOSED                                             1900-01-01 00:00:00.000

(1 row(s) affected)

-----------------------
2014-03-31 09:51:25.230

(1 row(s) affected)
&lt;/pre>
&lt;p></code></p>


<p>
  If you don't want to change the existing message exchange pattern then adding an explicit lifetime to <tt>BEGIN CONVERSATION</tt> is a viable solution. You have to be careful when choosing the lifetime, the "correct" value is very much application dependent. A lifetime declares that your application is no longer interested in delivering the messages sent after the lifetime has expired. Lifetime is per conversation, not per message. If the conversation is not complete (the two endpoints did not exchange <tt>EndDialog</tt> messages after the lifetime, then when the lifetime expires the conversation ends with error, an error message is en queued for your service. Choosing a lifetime like 60 seconds in my example is quite aggressive, specially in a distributed environment, it does not allow much room for retries if, for example, the target is going through some maintenance downtime. You may want to specify a lifetime of several hours, or maybe one day. Again, is always entirely application specific.
</p>


<h1>
  Solution 2: change the message pattern
</h1>


<p>
  The next solution I'll prpose is to change the actual message exchange pattern. Instead of closing the conversation from the initiator side immediately after the first <tt>SEND</tt>, let the target close the conversation when it receives the <tt>DEFAULT</tt> message. The initiator will in turn close its own endpoint in the activate procedure, as a result of receiving the <tt>EndDialog</tt> message. The setup is almost identical, with one minor different in target's activated procedure:
</p>


<p>
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Correct/target.sql">target.sql</a>
</p>


<pre><code class="prettyprint lang-sql">
create procedure usp_target
as
begin
	set nocount on;
	declare @h uniqueidentifier, @mt sysname;

	begin transaction;

	receive top(1) @mt = message_type_name, @h = conversation_handle from q;
	if (@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog' or
		@mt = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error' or 
		@mt = N'DEFAULT')
	begin
		end conversation @h;
	end
	commit
end
go
&lt;/pre>
&lt;p></code></p>


<p>
  And now when we send the message we no longer ending the dialog:
</p>


<p>
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Correct/dialog.sql">dialog.sql</a>
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
commit;
go
&lt;/pre>
&lt;p></code></p>


<p>
  After we let the conversation run its flow, we'll check the target endpoint:
</p>


<pre><code class="prettyprint lang-sql">
select  lifetime, state_desc, security_timestamp 
	from [target].sys.conversation_endpoints;

lifetime                state_desc                                  security_timestamp
----------------------- ------------------------------------------- -----------------------
2082-04-18 13:15:39.690 CLOSED                                      2014-03-31 10:32:11.307

(1 row(s) affected)
&lt;/pre>
&lt;p></code></p>


<p>
  Notice how with this message exchange pattern the target endpoint is in <tt>CLOSED</tt> state and <b>security timestamp is initialized</b>. The conversation endpoint will be removed by the system (deleted) when the security timestamp expires. Aside from fixing the endpoint leak problem, this pattern is better because it does have the <a href="http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/">>error handling related problems of the fire-and-forget pattern</a>.
</p>


<h1>
  The bad idea: WITH CLEANUP
</h1>


<p>
  All too often the team that notices the conversation leak problem comes up with a horrible 'fix': they add the <tt>WITH CLEANUP</tt> cause to the <tt>END CONVERSATION</tt> statement in the target activated procedure. A short test seems to show the problem was solved, but in a few days the same team ends up in Tier 3 Microsoft customer support calls reporting all sort of weird and strange problems, including multiple application logic processing of the same message (replays) and warnings logged in ERRORLOG complaining about database corruption (DBCC will not report any). This happens because WITH CLEANUP clause is a very rude way to end a conversation. The distributed state embedded in the conversation is abruptly ended, without any attempt to inform the peer endpoint. The complications that can arise are endless and I don't even know all of them. Basically, all bets are off.
</p>


<h1>
  Conclusion
</h1>


<p>
  The fire-and-forget message exchange pattern is all too compelling, but it is riddled with problems. Ending the conversation from the target side is a much better approach. Additionally consider <i>always</i> specifying a conversation lifetime, one that makes sense for whatever the message represents from your business point of views. A lifetime declares "if the messages were not delivered after this time, I'm no longer interested in trying to deliver them", so consider carefully what is a good value for your case. And never ever deploy <tt>WITH CLEANUP</tt> in production code.
</p>


<p>
  If you are already facing a situation in which the conversation endpoints have leaked then you first have to fix the application using one of the proposed solutions above (or both!). Then you can go ahead and cleanup the leaked conversations, and for that you can use <tt>END CONVERSATION ...WITH CLEANUP</tt>. Here is an example cleanup loop that looks for leaked endpoints and cleans them up:
</p>


<p>
  <a href="https://github.com/rusanu/conversation_endpoints_leaks/blob/master/Cleanup/cleanup.sql">cleanup.sql</a>
</p>


<pre><code class="prettyprint lang-sql">
declare @h uniqueidentifier, @count int = 0;
begin transaction;
while (1=1)
begin
	set @h = null;
	select top(1) @h = conversation_handle
		from sys.conversation_endpoints
		where state_desc = N'CLOSED'
		and security_timestamp = '1900-01-01 00:00:00.000'
		and lifetime > dateadd(month, 1, getutcdate())
	if (@h is null)
	begin
		break
	end
	end conversation @h with cleanup;
	set @count += 1;
	if (@count > 1000)
	begin
		commit;
		set @count = 0;
		begin transaction;
	end
end
commit
</code></pre>