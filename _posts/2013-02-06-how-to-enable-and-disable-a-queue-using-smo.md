---
id: 1713
title: How to enable and disable a queue using SMO
date: 2013-02-06T04:22:19+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=1713
permalink: /2013/02/06/how-to-enable-and-disable-a-queue-using-smo/
categories:
  - Samples
  - Tutorials
tags:
  - sharedmanagementobject
  - smo
---
The SMO object model for SQL Server [<tt>ServiceQueue</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.smo.broker.servicequeue.aspx) does allow one to enable or disable a queue, but the property that modifies the queue status is not intuitive, it is [<tt>IsEnqueueEnabled</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.smo.broker.servicequeue.isenqueueenabled.aspx):

> Gets or sets the Boolean property that specifies whether the queue is enabled.

This property matches the catalog view column <tt>is_enqueue_enabled</tt> in [<tt>sys.service_queues</tt>](http://msdn.microsoft.com/en-us/library/ms187795.aspx) but bears little resemblance to the T-SQL statement used to enable or disable a queue: <tt>ALTER QUEUE ... WITH STATUS = {ON|OFF}</tt>

<!--more-->

For example the following SMO code snippet:


<code class="prettyprint lang-sql">
&lt;pre>
            Server server = new Server("...");
            Database db = server.Databases["msdb"];
            ServiceQueue sq = new ServiceQueue(db.ServiceBroker, "foo");
            sq.Create();
            sq.IsEnqueueEnabled = false;
            sq.Alter();
            sq.IsEnqueueEnabled = true;
            sq.Alter();
&lt;/pre>
&lt;p></code>

generates the following T-SQL:


<code class="prettyprint lang-sql">
&lt;pre>
CREATE QUEUE [dbo].[foo];
ALTER QUEUE [dbo].[foo]  WITH STATUS = OFF ...;
ALTER QUEUE [dbo].[foo] WITH STATUS = ON ...;
&lt;/pre>
&lt;p></code>