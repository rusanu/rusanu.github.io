---
id: 27
title: The Mysterious Notification
date: 2006-06-17T15:49:55+00:00
author: remus
layout: post
guid: http://rusanu.com/2006/06/17/the-mysterious-notification/
permalink: /2006/06/17/the-mysterious-notification/
categories:
  - CodeProject
  - Tutorials
tags:
  - SqlDependency
---
**Update:**: for a way to leverage SqlDependency and Query Notifications from LINQ, see [SqlDependency based caching of LINQ Queries](http://rusanu.com/2010/08/04/sqldependency-based-caching-of-linq-queries/).

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  One of the most mysterious features shipped with SQL Server 2005 seems to be the various flavors of notifications on data change: <a href="http://msdn2.microsoft.com/en-us/system.data.sql.sqlnotificationrequest.aspx">SqlNotificationRequest</a>, <a href="http://msdn2.microsoft.com/en-us/system.data.sqlclient.sqldependency%28VS.80%29.aspx">SqlDependency</a> and <a href="http://msdn2.microsoft.com/en-us/system.web.caching.sqlcachedependency.aspx">SqlCacheDependency</a>. I see confusion on how this features work, how to use them and how to troubleshoot problems. Why are there three flavors of apparently the same functionality? And how is Service Broker involved into all of these?
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Query Notifications</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  In reality, there is only one feature in the SQL Server 2005 engine that delivers notifications on subscriptions for data changes: <a href="http://msdn2.microsoft.com/en-us/library/ms130764.aspx">Query Notifications</a>. Clients can submit a query requesting to be notified when data was modified in a manner that would change the query result and the server sends a notification when this change occurs. These requests are called ‘query notification subscriptions’. The list of notification subscriptions can be seen in the server level view <span style="font-family: 'Courier New'">sys.dm_qn_subscriptions</span>:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">select</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">*</span> <span style="color: blue">from</span> <span style="color: green">sys.dm_qn_subscriptions</span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Along with the query submitted for the notification, the client submits a service name and a broker instance. Each notification subscription begins a Service Broker dialog with this provided service and broker instance. When data is changed and the change would affect the result of the submitted query result, a message is sent on this dialog. By sending this message, the client is considered notified and the notification subscription is removed. If client desires to be notified on further changes, is supposed to subscribe again.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So we’ve seen that the notification is not delivered back to the client, but a Service Broker message is instead sent to the service the client provided in the subscription request. All normal rules for delivery, routing and dialog security apply to the dialog used to send this message. This means that the notification message can be sent to service hosted in the same database, in a different database or even on a remote machine. Also, there is no need for the client to be connected to receive the notification. It is perfectly acceptable for a client to submit a query for a notification subscription then disconnect and shutdown.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The client consumes the notification message just like any other Service Broker message: by RECEIVE-ing it from the service’s queue. The notification message will be of type <span style="font-family: 'Courier New'">[http://schemas.microsoft.com/SQL/Notifications/QueryNotification]</span>, an XML message type. This message is part of the <span style="font-family: 'Courier New'">[http://schemas.microsoft.com/SQL/Notifications/PostQueryNotification]</span>contract, which means that the service that receives the notification messages must be bound to this contract. After the client receives the message, is supposed to end the conversation on which the message was received, using END CONVERSATION (make sure you <a href="http://blogs.msdn.com/remusrusanu/archive/2006/01/27/518455.aspx">don’t use the CLEANUP</a> clause!).
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The restrictions that apply to queries submitted for notifications are explained in BOL in the ‘Creating a Query for Notification’ topic:<a href="http://msdn2.microsoft.com/en-us/library/ms181122%28SQL.90%29.aspx">http://msdn2.microsoft.com/en-us/library/ms181122(SQL.90).aspx</a>. Although the query can be a stored procedure, it must not contain any flow control statements (IF, WHILE, BEGIN TRY<span>  </span>etc).
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Clients can submit notification subscription request by programming directly against the SQL Native Client, using the HTTP SOAP access to SQL Server or, most commonly, using the ADO.Net client components. AFAIK clients cannot subscribe using the OLEDB, ODBC or JDBC components. <b>Updated:</b> It is possible from OleDB and ODBC as well, see <a href="http://msdn.microsoft.com/en-us/library/ms130764.aspx" target="_blank">Working with Query Notifications</a>.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">The cost of a subscription</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Query notification subscriptions have little cost in the SQL Server 2005 engine. Notification subscriptions are only metadata, and the effect of a subscription is to modify the query plans in a manner that allows the relevant data changes to be detected. The picture below shows how a plan for a INSERT INTO TEST … statement is modified when there is a query notification subscription active with the query SELECT * FROM TEST:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  &nbsp;
</p>

<a href="http://blogs.msdn.com/photos/remusrusanu/images/635600/original.aspx" target="_blank"><img src="http://blogs.msdn.com/photos/remusrusanu/images/635600/secondarythumb.aspx" border="0" /></a>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This plan shows that, while cheap, query notification subscriptions are not free. The cost associated with them is similar to the cost of having a secondary index on the data, or the cost of having an indexed view.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">Server Restart</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  When the SQL Server 2005 is restarted, all query notification subscriptions are notified and ended.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">KILL QUERY NOTIFICATION SUBSCRIPTION</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This Transact-SQL statement can be used to administratively end query notification subscriptions. The BOL describe its usage: <a href="http://msdn2.microsoft.com/en-us/library/ms186764.aspx">http://msdn2.microsoft.com/en-us/library/ms186764.aspx</a>
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">SqlNotificationRequest</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This is simplest ADO.Net component for subscribing to Query Notifications. This class is used directly to create a query notifications subscription. The usage is straightforward:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>Create a new SqlNotificationRequest object, passing in the appropriate Service Broker service name and broker instance
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>Assign the newly created SqlNotificationRequest to the Notification property of a SqlCommand
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>Execute the SqlCommand.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  MSDN has a sample of this technique here: <a href="http://msdn2.microsoft.com/en-us/3ht3391b.aspx">http://msdn2.microsoft.com/en-us/3ht3391b.aspx</a>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Using the SqlNotificationRequest leaves the task of handling the notification message entirely to the client application. While this is the most flexible way of leveraging the Query Notification functionality, it requires knowledge and understanding of the way Service Broker delivers the messages and how to write a Service Broker application to process the notification messages.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">SqlDependency</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This component tries to make the task of handling the Query Notification subscription messages as straightforward as possible. Using the SqlDependency the application developer gets a CLR callback whenever data has changed. How this is achieved is very simple: the SqlDependency uses SqlNotification to subscribe to query notifications and then the SqlDependency infrastructure uses a just-in-time-created service and queue to receive the notification messages. It starts a background client thread that posts a WAITFOR(RECEIVE…) on this queue and whenever a notification message is delivered by Service Broker into this queue, the background thread will receive this message, inspects it and invoke the appropriate callback. The static methods <a href="http://msdn2.microsoft.com/en-us/system.data.sqlclient.sqldependency.start.aspx">Start</a> and <a href="http://msdn2.microsoft.com/en-us/system.data.sqlclient.sqldependency.stop.aspx">Stop</a> on the SqlDependency class are starting and stopping this background thread, as well as create and drop the service and queue used by the SqlDependency infrastructure. This background thread is shared by all requests in one appdomain. This is important, because if each SqlDependency request would start its own listener, the back end server would quickly get swamped by all those requests issuing WAITFOR(RECEIVE…) statements, each blocking a server thread.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The MSDN contains a sample of how to use the SqlDependency here: <a href="http://msdn2.microsoft.com/en-us/a52dhwx7.aspx">http://msdn2.microsoft.com/en-us/a52dhwx7.aspx</a>. Note how, similarly to the SqlNotification usage, the client is expected to subscribe again if it whishes to be further notified.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">Abrupt client disconnects</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  If a client disconnects abruptly without stopping the SqlDependency infrastructure, the queue and service created by the SqlDependency.Start are left stranded. This is not a problem, because the SqlDependency infrastructure also deploys a stored proc associated with activation on this queue and sets up a <a href="http://msdn2.microsoft.com/en-us/library/ms187804.aspx">dialog timer</a>. When this timer fires, the procedure is activated and this procedure cleans up the queue, the service and the activated procedure itself. Due to the pending WAITFOR(RECEIVE…) permanently posted by the SqlDependency background thread, the activated procedure will launch only if the client has disconnected w/o cleaning up.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">SqlCacheDependency</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This is a component designed specifically for ASP.Net to cache data, using the SqlDependency whenever possible. I’m not going to dwell into how it works, simply because I don’t really know. But once one understands how the Query Notifications and SqlDependency class work, the SqlCacheDependency is just another use of this infrastructure, with nothing special to it.
</p>

<h1 style="margin: 12pt 0in 3pt">
  <font face="Arial" size="5">Troubleshooting Query Notifications</font>
</h1>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  While the SqlDependency infrastructure is a great help to developers, is it often used w/o properly understanding its functionality and I often see people totally lost when it comes to troubleshooting a problem. In fact, BOL has a dedicated chapter for this topic here: <a href="http://msdn2.microsoft.com/en-us/library/ms177469.aspx">http://msdn2.microsoft.com/en-us/library/ms177469.aspx</a>. The Profiler can show the Query Notification events that are reported when a new subscription is registered. Once a notification subscription is notified, the notification message is delivered using Service Broker and all of my comments related to <a href="http://rusanu.com/2005/12/20/troubleshooting-dialogs/">troubleshooting dialogs</a> apply to this message delivery as well. If the notification message is no delivered, the first place to look is the transmission_status column in the sys.trasnmission_queue view in the sender’s database.
</p>