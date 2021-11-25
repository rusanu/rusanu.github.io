---
id: 1718
title: How to enable and disable a queue using SMO
date: 2013-02-06T04:21:05+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/02/06/1713-revision-3/
permalink: /2013/02/06/1713-revision-3/
---
The SMO object model for SQL Server [<tt>ServiceQueue</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.smo.broker.servicequeue.aspx) does allow one to enable or disable a queue, but the property that modifies the queue status is not intuitive: [<tt>IsEnqueueEnabled</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.management.smo.broker.servicequeue.isenqueueenabled.aspx):

> Gets or sets the Boolean property that specifies whether the queue is enabled.

This property matches the catalog view column <tt>is_enqueue_enabled</tt> in [<tt>sys.service_queues</tt>](http://msdn.microsoft.com/en-us/library/ms187795.aspx) but bears little resemblance to the T-SQL stateemnt used to enable or disable a queue: <tt>ALTER QUEUE ... WITH STATUS = {ON|OFF}</tt>

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