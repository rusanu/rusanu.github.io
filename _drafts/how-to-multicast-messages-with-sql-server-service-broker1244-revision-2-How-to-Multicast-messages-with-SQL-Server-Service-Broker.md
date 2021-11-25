---
id: 1246
title: How to Multicast messages with SQL Server Service Broker
date: 2011-07-20T10:00:30+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/20/1244-revision-2/
permalink: /2011/07/20/1244-revision-2/
---
Starting with SQL Server 11 the the <a href="http://msdn.microsoft.com/en-us/library/ms188407%28v=SQL.110%29.aspx"target="_blank"><tt>SEND</tt></a> verb has a new syntax and accepts multiple dialogs handles to send on:

<pre>SEND
   ON CONVERSATION [(]conversation_handle [,.. @conversation_handle_n][)]
   [ MESSAGE TYPE message_type_name ]
   [ ( message_body_expression ) ]
[ ; ]
</pre></p>