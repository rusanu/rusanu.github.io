---
id: 1042
title: How to pass a NULL value in a message to a queue in SQL Server
date: 2011-01-15T10:00:58+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/01/15/1037-revision-4/
permalink: /2011/01/15/1037-revision-4/
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