---
id: 72
title: When it rains, it pours
date: 2007-11-10T20:22:55+00:00
author: remus
layout: post
guid: /2007/11/10/when-it-rains-it-pours/
permalink: /2007/11/10/when-it-rains-it-pours/
categories:
  - Troubleshooting
tags:
  - SqlDependency
---
I’ve seen a number of customers reporting problems about ERRORLOG growing out of control (tens of GBs) because of error like following:

<pre class="errorlog">2007-10-12 11:18:32.44 spid25s The query notification dialog on conversation handle ‘{EC54573A-9978-DC11-961C-00188B111155}.’ closed due to the following error: ‘&lt;?xml version=”1.0??>&lt;Error xmlns=”http://schemas.microsoft.com/SQL/ServiceBroker/Error”>&lt;Code>-8490&lt;/Code>&lt;Description>Cannot find the remote service ‘SqlQueryNotificationService-4869b411-fa1c-4d8a-ab37-5bf5762eb98b’ because it does not exist.&lt;/Description>&lt;/Error>’.

2007-10-12 11:37:20.69 spid51s The activated proc [dbo].[ SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d] running on queue tempdb.dbo. SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d output the following: ‘Could not find stored procedure ‘dbo. SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d‘

2007-10-12 10:59:39.32 spid51 Error: 28054, Severity: 11, State: 1.
2007-10-12 10:59:39.32 spid51 Service Broker needs to access the master key in the database ‘tempdb’. Error code:25. The master key has to exist and the service master key encryption is required.
</pre>

All these messages are related one way or another to the ADO.Net component SqlDependency. I’ll present each one how it is caused and how to avoid id.

<!--more-->

First, lets review in brief how the SqlDependency works. The application is supposed to invoke the static method SqlDependency.Start at startup to deploy the necessary infrastructure, then use instances of SqlDependency object associated with a SqlCommand to receive callbacks when the query executed is notified (data has changed), and finally call SqlDependency.Stop when the application shuts down to tear down the infrastructure deployed at startup. I have explained before how the server side Query Notifications feature works to detect the changes and to notify the subscriptions, see </2006/06/17/the-mysterious-notification>.

The three errors above are all a result of the way how SqlDependency deploys and cleans up it’s infrastructure. The first error happens in the following scenario:

  1. SqlDependency.Start () is invoked by an application. At this moment a service, a queue and a procedure are created.
  2. SqlDependency is used to subscribe to query notifications. Perhaps some queries are notified and re-subscribed, in a normal operations mode.
  3. Application exists, SqlDependency.Stop is called and the service, queue and procedure are dropped. However, there are still subscribed notifications pending on the server.
  4. One or more of the pending subscriptions are notified in the server. This cause a notification message to be sent, but the destination service was dropped, so an error is returned to the sender. The QN receives this error and displays an error message in the ERRORLOG: <pre class="errorlog">2007-10-12 11:18:32.44 spid25s The query notification dialog on conversation handle ‘{EC54573A-9978-DC11-961C-00188B111155}.’ closed due to the following error: ‘&lt;?xml version=”1.0″?>&lt;Error xmlns=”http://schemas.microsoft.com/SQL/ServiceBroker/Error”>&lt;Code>-8490&lt;/Code>&lt;Description>Cannot find the remote service ‘SqlQueryNotificationService-4869b411-fa1c-4d8a-ab37-5bf5762eb98b’ because it does not exist.&lt;/Description>&lt;/Error>’
</pre></ol> 
    
    This is straightforward scenario and the application developer has done no mistake. However, this case should not result in huge ERRORLOG growth, because it will happen only for subscriptions notified after an application had exited. So if an application runs twice for 4 hours a day and subscribes to 10 queries, it should cause at most 20 entries like this a day. Multiplied by a decent size deployment base you will get something like 2000 entries for a 100 users deployment, annoying but not fatal. If you find your ERROLOG swamped by the message above (I’ve heard of thousands of entries added per hour), review your application behavior. Most likely you are calling SqlDepdendency.Stop way too often. Normally it should be called only on AppDomain unload. A possible workaround would probably have to be based on the kill query notification subscription verb, forcing the application to cleanup any pending subscription at shutdown. Good luck figuring out your own subscriptions from the other instances of the same application…
    
    The second error message is a bit trickier. This one happens because the SqlDepndency cleanup attempted to drop the service, queue and procedure. Here is an example of how the cleanup procedure looks like:
    
    <pre><code class="prettyprint lang-sql">
CREATE PROCEDURE [SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d]
AS
BEGIN
BEGIN TRANSACTION;
RECEIVE TOP(0)
conversation_handle
FROM [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d];
IF (SELECT COUNT(*) FROM [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d]
WHERE message_type_name = ‘http://schemas.microsoft.com/SQL/ServiceBroker/DialogTimer’) > 0
BEGIN
DROP SERVICE [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d];
DROP QUEUE [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d];
DROP PROCEDURE [SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d];
END
COMMIT TRANSACTION;
END
</code></pre>
    
    It is possible that for whatever reason it fails to drop the service and the queue, but it dropped the procedure. Since there is no referential integrity rule to prevent the drop of a procedure that is attached as activation procedure to a queue, it is possible to drop the procedure and leave the queue declared with activation pointing to a missing procedure. In this case the activation mechanism will trigger the error message every five seconds:
    
    <pre class="errorlog">2007-10-12 11:37:20.69 spid51s The activated proc [dbo].[ SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d] running on queue tempdb.dbo. SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d output the following: ‘Could not find stored procedure ‘dbo. SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d‘
</pre>
    
    Every five seconds for each queue left behind by SqlDependency, now that will grow the ERRORLOG to fill every last sector available on the disk in no time at all. The queues that have this problem must be manually fixed (ie. DROP QUEUE). One can identify them by looking up sys.service_queues for queues that have an activation procedure that no longer exists. Also one must identify the cause of the problem, why the service and queue could not be dropped, since the application that is using SqlDependency will continue to create new queues with orphaned activation.
    
    Finally the last problem is the message about the lack of a master key in the database:
    
    <pre class="errorlog">2007-10-12 10:59:39.32 spid51 Error: 28054, Severity: 11, State: 1.
2007-10-12 10:59:39.32 spid51 Service Broker needs to access the master key in the database ‘tempdb’. Error code:25. The master key has to exist and the service master key encryption is required.
</pre>
    
    This message is usually also caused by SqlDependency because the timer it creates to cleanup the service, queue and procedure needs a conversation (to use BEGIN CONVERSATION TIMER). This conversation is started without specifying the ENCRYPTION = OFF clause, so it will need a database master key to store the generated session keys. Although this conversation never sends any message and session keys are not actually needed, the message is periodically logged into the ERRORLOG. To avoid this message, there is a trivial workaround: simply create a database master key in the database