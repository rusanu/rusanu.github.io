---
id: 31
title: Consuming Event Notifications from CLR
date: 2007-04-11T20:11:00+00:00
author: remus
layout: post
guid: /2007/04/11/consuming-event-notifications-from-clr/
permalink: /2007/04/11/consuming-event-notifications-from-clr/
categories:
  - Samples
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">A question was asked on the newsgroups: how can a C# program be notified when the schema of a table was modified (i.e. a column was added)? The short answer is to use Event Notifications, see </font><a href="http://msdn2.microsoft.com/en-us/library/ms189453.aspx"><font color="#800080" face="Times New Roman" size="3">http://msdn2.microsoft.com/en-us/library/ms189453.aspx</font></a>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font size="3"><font face="Times New Roman">But an example would help </font><span style="font-family: Wingdings"><span>J</span></span><font face="Times New Roman">. First thing, we need to set up a queue and service that will receive the notifications:</font></font><!--more-->
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [myapplicationdatabase]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [NotificationsQueue]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">SERVICE</span> [MyNotificationsService] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">QUEUE</span> [NotificationsQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Next we create an Event Notification for the ALTER_TABLE event, asking the notifications to be sent to our newly created service:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [myapplicationdatabase]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">EVENT</span> <span style="color: blue">NOTIFICATION</span> [NotificationAlterTable]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">DATABASE<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> ALTER_TABLE<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> <span style="color: red">&#8216;MyNotificationsService&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;current database&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">With this, we had set in place the infrastructure to be notified when a table is altered. We could had choose any other event from the many notification types supported by Event Notifications, like other DDL events (CREATE, DROP) or even trace events (e.g. LOCK_DEADLOCK to be notified when a deadlock occurs). Whenever the notification is fired, a message will be enqueued in the </font><span style="font-size: 10pt; font-family: 'Courier New'">[NotificationsQueue] </span><font face="Times New Roman" size="3">queue.</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">How does a C# application consume those messages? Using the RECEIVE verb. This verb produces a result set just like a SELECT, so messages are being consumed by creating and iterating a data reader. Here is a dummy example of a C# program that will receive our notifications and display them on the console:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Collections.Generic;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Text;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Data.SqlClient;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Data;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Data.SqlTypes;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">using</span><span style="font-size: 10pt; font-family: 'Courier New'"> System.Xml;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">namespace</span><span style="font-size: 10pt; font-family: 'Courier New'"> ReceiveEventNotifications<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">class</span> <span style="color: teal">Program<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">const</span> <span style="color: blue">string</span> _sqlReceive =<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: maroon; font-family: 'Courier New'">@&#8221;WAITFOR (RECEIVE <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: maroon; font-family: 'Courier New'"><span> </span>conversation_handle, <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: maroon; font-family: 'Courier New'"><span> </span>message_type_name,<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: maroon; font-family: 'Courier New'"><span> </span>CAST(message_body as XML)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: maroon; font-family: 'Courier New'"><span> </span>FROM [NotificationsQueue]);&#8221;</span><span style="font-size: 10pt; font-family: 'Courier New'">;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">const</span> <span style="color: blue">string</span> _sqlEndDialog = <span style="color: maroon">@&#8221;END CONVERSATION @conversationHandle&#8221;</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">const</span> <span style="color: blue">string</span> _errorMessageType = <span style="color: maroon">@&#8221;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8221;</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">const</span> <span style="color: blue">string</span> _endDialogMessageType = <span style="color: maroon">@&#8221;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8221;</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">static</span> <span style="color: blue">void</span> <st1:place w:st="on">Main</st1:place>(<span style="color: blue">string</span>[] args)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: teal">SqlConnectionStringBuilder</span> scsb = <span style="color: blue">new</span> <span style="color: teal">SqlConnectionStringBuilder</span>();<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>scsb.DataSource = <span style="color: maroon">&#8220;.&#8221;</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>scsb.IntegratedSecurity = <span style="color: blue">true</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>scsb.InitialCatalog = <span style="color: maroon">&#8220;target&#8221;</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>scsb.MultipleActiveResultSets = <span style="color: blue">true</span>;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">using</span> (<span style="color: teal">SqlConnection</span> conn = <span style="color: blue">new</span> <span style="color: teal">SqlConnection</span>(scsb.ConnectionString))<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conn.Open();<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: teal">SqlCommand</span> cmdReceive = <span style="color: blue">new</span> <span style="color: teal">SqlCommand</span>(_sqlReceive, conn);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">// Set the command timeout to infinite<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>cmdReceive.CommandTimeout = 0;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">while</span> (<span style="color: blue">true</span>)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">using</span> (<span style="color: teal">SqlTransaction</span> trn = conn.BeginTransaction())<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>cmdReceive.Transaction = trn;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">using</span> (<span style="color: teal">SqlDataReader</span> rdr = cmdReceive.ExecuteReader(<span style="color: teal">CommandBehavior</span>.SequentialAccess))<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">while</span> (rdr.Read())<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: teal">Guid</span> conversationHandle = rdr.GetGuid(0);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">string</span> messageType = rdr.GetString(1);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">if</span> (messageType == _endDialogMessageType ||</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>messageType == _errorMessageType)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: teal">SqlCommand</span> cmdEndDialog = <span style="color: blue">new</span> <span style="color: teal">SqlCommand</span>(_sqlEndDialog, conn, trn);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>cmdEndDialog.Parameters.AddWithValue(<span style="color: maroon">&#8220;@conversationHandle&#8221;</span>, conversationHandle);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>cmdEndDialog.ExecuteNonQuery();<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">else<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>{<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: teal">SqlXml</span> xmlNotification = rdr.GetSqlXml(2);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: teal">XmlDocument</span> doc = <span style="color: blue">new</span> <span style="color: teal">XmlDocument</span>();<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>doc.Load(xmlNotification.CreateReader());<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: teal">Console</span>.WriteLine(doc.OuterXml);<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>trn.Commit();<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">}<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Arial">Things worthy of note:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; font-family: Arial"><span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; font-family: Arial">the infinite CommandTimeout (since </span><span style="font-size: 10pt; font-family: 'Courier New'">WAITFOR</span><span style="font-size: 10pt; font-family: Arial"> will block until a message arrives)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; font-family: Arial"><span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; font-family: Arial">the use of </span><span style="font-size: 10pt; font-family: 'Courier New'">SqlXml.CreateReader</span><span style="font-size: 10pt; font-family: Arial"> to get the message payload as an </span><span style="font-size: 10pt; font-family: 'Courier New'">XmlReader</span><span style="font-size: 10pt; font-family: Arial"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; font-family: Arial"><span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; font-family: Arial">the use of a </span><span style="font-size: 10pt; font-family: 'Courier New'">SequentialAccess</span><span style="font-size: 10pt; font-family: Arial"> command behavior to optimize the </span><span style="font-size: 10pt; font-family: 'Courier New'">XmlReader</span><span style="font-size: 10pt; font-family: Arial"> stream <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; font-family: Arial"><span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; font-family: Arial">the need for </span><span style="font-size: 10pt; font-family: 'Courier New'">MultipleActiveResultSets</span><span style="font-size: 10pt; font-family: Arial"> (aka MARS) to be able to respond with </span><span style="font-size: 10pt; font-family: 'Courier New'">END CONVERSATION</span><span style="font-size: 10pt; font-family: Arial"> to incomming error and end messages while the current connection has an active result set<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; font-family: Arial"><span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; font-family: Arial">the processing of EndDialog and Error message types by ending our side of the conversation<o:p></o:p></span>
</p>

<span style="font-size: 10pt; font-family: Arial">One thing to keep in mind is that the Event Notifications are going to be fired even when the application is not running. So the application might start up and find a number of notifications pending from the period was not running and has to be prepared to deal with this case.</span>