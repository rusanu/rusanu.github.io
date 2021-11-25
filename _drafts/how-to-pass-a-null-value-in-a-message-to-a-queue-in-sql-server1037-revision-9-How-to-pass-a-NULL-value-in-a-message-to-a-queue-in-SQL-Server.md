---
id: 2048
title: How to pass a NULL value in a message to a queue in SQL Server
date: 2011-01-15T10:08:51+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/01/15/1037-revision-9/
permalink: /2011/01/15/1037-revision-9/
---
The <a href="http://msdn.microsoft.com/en-us/library/ms188407.aspx" target="_blank">SEND</a> Transact-SQL verb does not allow to send a NULL message body, attempting to do so will result in error:

<pre style="color:red;"><code class="prettyprint lang-sql">
Msg 8433, Level 16, State 1, Line 11
The message body may not be NULL.  A zero-length UNICODE or binary string is allowed.</code>
</pre>

But there are ways to send a NULL message body. One way is to completely omit the message body argument:

<pre><code class="prettyprint lang-sql">
SEND ON CONVERSATION @handle MESSAGE TYPE [...];
</code>
</pre>

Another way is to send a 0 length message body, which will be enqueued as a NULL message body in the target queue:

<pre><code class="prettyprint lang-sql">
SEND ON CONVERSATION @handle MESSAGE TYPE [...] (0x);
SEND ON CONVERSATION @handle MESSAGE TYPE [...] ('');
SEND ON CONVERSATION @handle MESSAGE TYPE [...] (N'');
</code>
</pre>

All three forms above will enqueue the same message body: NULL. This is true for both binary messages (<tt>VALIDATION = NONE</tt>) and for XML messages (<tt>VALIDATION=WELL_FORMED_XML</tt>).

Here is a short test script showing this:

<pre><code class="prettyprint lang-sql">
create message type [BINARY] validation = none;
create message type [XML] validation = well_formed_xml;
go

create contract [TEST] (
	[BINARY] sent by initiator,
	[XML] sent by initiator);
go

create queue Sender;
create service Sender on queue Sender;
go

create queue Receiver;
create service Receiver on queue Receiver  ([TEST]);
go

declare @h uniqueidentifier;

begin dialog conversation @h
from service [Sender]
to service N'Receiver', N'current database'
on contract [TEST]
with encryption = off;

send on conversation @h message type [BINARY];
send on conversation @h message type [BINARY] (0x);

send on conversation @h message type [XML];
send on conversation @h message type [XML] ('');
send on conversation @h message type [XML] (N'');

receive * from [Receiver];
go
&lt;/pre>
&lt;p></code></p>


<p>
  The received <tt>message_body</tt> column have a NULL value for all 5 messages sent.
</p>