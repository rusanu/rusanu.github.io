---
id: 417
title: When it rains, it pours
date: 2008-10-26T07:34:24+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/10/26/72-revision-2/
permalink: /2008/10/26/72-revision-2/
---
<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">I’ve seen a number of customers reporting problems about ERRORLOG growing out of control (tens of GBs) because of error like following:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <span style="font-size: 9pt; font-family: 'Courier New'">2007-10-12 11:18:32.44 spid25s<span> </span>The query notification dialog on conversation handle &#8216;{EC54573A-9978-DC11-961C-00188B111155}.&#8217; closed due to the following error: &#8216;<?xml version=&#8221;1.0&#8243;?><Error xmlns=&#8221;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8221;><Code>-8490</Code><Description>Cannot find the remote service &#8216;SqlQueryNotificationService-4869b411-fa1c-4d8a-ab37-5bf5762eb98b&#8217; because it does not exist.</Description></Error>&#8217;.<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <span style="font-size: 9pt; font-family: 'Courier New'">2007-10-12 11:37:20.69 spid51s<span> </span>The activated proc [dbo].[<span> SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d</span>] running on queue tempdb.dbo.<span> SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d</span> output the following:<span> </span>&#8216;Could not find stored procedure &#8216;dbo.<span> SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d</span>&#8216;<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <span style="font-size: 9pt; font-family: 'Courier New'">2007-10-12 10:59:39.32 spid51<span> </span>Error: 28054, Severity: 11, State: 1.<br /> 2007-10-12 10:59:39.32 spid51<span> </span>Service Broker needs to access the master key in the database &#8216;tempdb&#8217;. Error code:25. The master key has to exist and the service master key encryption is required.<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">All these messages are related one way or another to the ADO.Net component SqlDependency.<span> </span>I’ll present each one how it is caused and how to avoid id.</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">First, lets review in brief how the SqlDependency works. The application is supposed to invoke the static method </font><a href="http://msdn2.microsoft.com/en-us/library/system.data.sqlclient.sqldependency.start.aspx"><font face="Calibri" size="3">SqlDependency.Start</font></a><font face="Calibri" size="3"> at startup to deploy the necessary infrastructure, then use instances of </font><a href="http://msdn2.microsoft.com/en-us/library/t9x04ed2.aspx"><font face="Calibri" size="3">SqlDependency</font></a><font face="Calibri" size="3"> object associated with a SqlCommand to receive callbacks when the query executed is notified (data has changed), and finally call </font><a href="http://msdn2.microsoft.com/en-us/library/system.data.sqlclient.sqldependency.stop.aspx"><font face="Calibri" size="3">SqlDependency.Stop</font></a><font face="Calibri" size="3"> when the application shuts down to tear down the infrastructure deployed at startup. I have explained before how the server side Query Notifications feature works to detect the changes and to notify the subscriptions, see </font><a href="http://blogs.msdn.com/remusrusanu/archive/2006/06/17/635608.aspx"><font face="Calibri" size="3">http://blogs.msdn.com/remusrusanu/archive/2006/06/17/635608.aspx</font></a>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">The three errors above are all a result of the way how SqlDependency deploys and cleans up it’s infrastructure. The first error happens in the following scenario:</font>
</p>

<p class="ListParagraphCxSpFirst" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in; line-height: normal">
  <span><span><font face="Calibri" size="3">1)</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Calibri" size="3">SqlDependency.Start () is invoked by an application. At this moment a service, a queue and a procedure are created.</font>
</p>

<p class="ListParagraphCxSpMiddle" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in; line-height: normal">
  <span><span><font face="Calibri" size="3">2)</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Calibri" size="3">SqlDependency is used to subscribe to query notifications. Perhaps some queries are notified and re-subscribed, in a normal operations mode</font>
</p>

<p class="ListParagraphCxSpMiddle" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in; line-height: normal">
  <span><span><font face="Calibri" size="3">3)</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Calibri" size="3">Application exists, SqlDependency.Stop is called and the service, queue and procedure are dropped.<span> </span>However, there are still subscribed notifications pending on the server. </font>
</p>

<p class="ListParagraphCxSpLast" style="margin: 0in 0in 10pt 0.5in; text-indent: -0.25in; line-height: normal">
  <span style="color: red"><span><font face="Calibri" size="3">4)</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Calibri" size="3">One or more of the pending subscriptions are notified in the server. This cause a notification message to be sent, but the destination service was dropped, so an error is returned to the sender. The QN receives this error and displays an error message in the ERRORLOG:<br /> </font><span style="font-size: 9pt; color: red; font-family: 'Courier New'">2007-10-12 11:18:32.44 spid25s<span> </span>The query notification dialog on conversation handle &#8216;{EC54573A-9978-DC11-961C-00188B111155}.&#8217; closed due to the following error: &#8216;<?xml version=&#8221;1.0&#8243;?><Error xmlns=&#8221;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8221;><Code>-8490</Code><Description>Cannot find the remote service &#8216;SqlQueryNotificationService-4869b411-fa1c-4d8a-ab37-5bf5762eb98b&#8217; because it does not exist.</Description></Error>&#8217;</span><span style="color: red"><o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">This is straightforward scenario and the application developer has done no mistake. However, this case should not result in huge ERRORLOG growth, because it will happen only for subscriptions notified after an application had exited. So if an application runs twice for 4 hours a day and subscribes to 10 queries, it should cause at most 20 entries like this a day. Multiplied by a decent size deployment base you will get something like 2000 entries for a 100 users deployment, annoying but not fatal. If you find your ERROLOG swamped by the message above (I’ve heard of thousands of entries added per hour), review your application behavior. Most likely you are calling SqlDepdendency.Stop way too often. Normally it should be called only on AppDomain unload. A possible workaround would probably have to be based on the </font><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">kill</span><span style="font-size: 10pt; font-family: 'Courier New'"> query <span style="color: blue">notification</span> subscription </span><font face="Calibri" size="3">verb, forcing the application to cleanup any pending subscription at shutdown. Good luck figuring out your own subscriptions from the other instances of the same application…</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">The second error message is a bit trickier. This one happens because the SqlDepndency cleanup attempted to drop the service, queue and procedure. Here is an example of how the cleanup procedure looks like:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d] <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS</span><span style="font-size: 10pt; font-family: 'Courier New'"> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span><span style="color: gray">(</span><span style="color: gray">)</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">conversation_handle</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d]<span style="color: gray">;<o></o></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: blue">SELECT</span> <span style="color: fuchsia">COUNT</span><span style="color: gray">(*)</span> <span style="color: blue">FROM</span> [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d] <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> message_type_name <span style="color: gray">=</span> <span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/DialogTimer&#8217;</span><span style="color: gray">)</span> <span style="color: gray">></span> 0 <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">SERVICE</span> [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d]<span style="color: gray">;</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">QUEUE</span> [SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d]<span style="color: gray">;</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d]<span style="color: gray">;</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt; line-height: normal">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;</span> <o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <o><font face="Calibri" size="3"> </font></o>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt; line-height: normal">
  <font face="Calibri" size="3">It is possible that for whatever reason it fails to drop the service and the queue, but it dropped the procedure. Since there is no referential integrity rule to prevent the drop of a procedure that is attached as activation procedure to a queue, it is possible to drop the procedure and leave the queue declared with activation pointing to a missing procedure. In this case the activation mechanism will trigger the error message every five seconds:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt">
  <span style="font-size: 9pt; color: red; line-height: 115%; font-family: 'Courier New'">2007-10-12 11:37:20.69 spid51s<span> </span>The activated proc [dbo].[<span> SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d</span>] running on queue tempdb.dbo.<span> SqlQueryNotificationService-633a0c13-66e4-410e-8bd8-744146d2258d</span> output the following:<span> </span>&#8216;Could not find stored procedure &#8216;dbo.<span> SqlQueryNotificationStoredProcedure-633a0c13-66e4-410e-8bd8-744146d2258d</span>&#8216;<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt">
  <span style="font-size: 9.5pt; line-height: 115%; font-family: Arial">Every five seconds for each queue left behind by SqlDependency, now that will grow the ERRORLOG to fill every last sector available on the disk in no time at all. The queues that have this problem must be manually fixed (ie. DROP QUEUE). One can identify them by looking up sys.service_queues for queues that have an activation procedure that no longer exists. Also one must identify the cause of the problem, why the service and queue could not be dropped, since the application that is using SqlDependency will continue to create new queues with orphaned activation.<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt">
  <span style="font-size: 9.5pt; line-height: 115%; font-family: Arial">Finally the last problem is the message about the lack of a master key in the database:<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt">
  <span style="font-size: 9.5pt; color: red; line-height: 115%; font-family: Arial">2007-10-12 10:59:39.32 spid51 Error: 28054, Severity: 11, State: 1.<br /> 2007-10-12 10:59:39.32 spid51 Service Broker needs to access the master key in the database &#8216;tempdb&#8217;. Error code:25. The master key has to exist and the service master key encryption is required.<o></o></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 10pt">
  <span style="font-size: 9.5pt; line-height: 115%; font-family: Arial">This message is usually also caused by SqlDependency because the timer it creates to cleanup the service, queue and procedure needs a conversation (to use BEGIN CONVERSATION TIMER). This conversation is started without specifying the ENCRYPTION = OFF clause, so it will need a database master key to store the generated session keys. Although this conversation never sends any message and session keys are not actually needed, the message is periodically logged into the ERRORLOG. To avoid this message, there is a trivial workaround: simply create a database master key in the database</span>
</p>