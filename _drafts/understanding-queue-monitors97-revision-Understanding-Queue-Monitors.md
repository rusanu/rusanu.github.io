---
id: 1168
title: Understanding Queue Monitors
date: 2008-08-03T17:58:04+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/08/03/97-revision/
permalink: /2008/08/03/97-revision/
---
A major class of Service Broker applications have nothing to do with the distributed application capabilities SSB was designed for. Instead they leverage SSB&#8217;s capability to run a stored procedure outside the context of the user connection. This capability enables to schedule execution and don&#8217;t wait for it to finish, or to schedule multiple procedures to execute in paralel. Often the developers that use these capabilities don&#8217;t care for the reliable message delivery SSB offers and many see the complex infrastructure of message types, contracts and services as a mere hurdle. Nonetheless sooner or later they have to troubleshoot an issue and then is when many of you find my blog and my articles. Troubleshooting activated procedure is not difficult, but one simply has to know where to look and what to look for.

In this entry I want to shed some light in the soul that drives the activation process: the Queue Monitors.

<!--more-->

A Queue Monitor is a state machine that resides in the SQL Server and decides when to launch a new instance of activated procedure into execution. The <code class="prettyprint lang-sql">&lt;span style="color: green">sys.dm_broker_queue_monitors&lt;/span></code> DMV is used to view the Queue Monitors:

<div class="post-image">
  <a href="http://test.rusanu.com/wp-content/uploads/2008/08/dm_broker_queue_monitors.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/08/dm_broker_queue_monitors.png" alt="select * from sys.dm_broker_queue_monitors" title="Click on the image for a full size view" /></a>
</div>

This DMV is described in BOL at <a href="http://msdn.microsoft.com/en-us/library/ms177628.aspx" target="_blank">http://msdn.microsoft.com/en-us/library/ms177628.aspx</a> where the columns types and decriptions can be found.

There is a Queue Monitor for each queue with activation enabled in the SQL Server instance. The DMV does not contain the database name nor the queue name, but you can join with <code class="prettyprint lang-sql">&lt;span style="color: green">sys.service_queues&lt;/span></code> on queue_id to get the queue names. Note that even when you run this query on a brand new install of SQL Server you will get two Queue Monitors: <code class="prettyprint lang-sql">InternalMailQueue</code> and <code class="prettyprint lang-sql">ExternalMailQueue</code>. These are the queues placed in <code class="prettyprint lang-sql">msdb</code> used by the Database Mail feature of SQL Server.

## Queue Monitor States

To start troublesooting an activation problem you need to understand the Queue Monitor states:

  * <code class="prettyprint lang-sql">&lt;span style="color: Green">INACTIVE&lt;/span></code>: there are no messages available to be received in the monitored queue.
  * <code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code>: messages are available and the activated procedure was launched into execution.
  * <code class="prettyprint lang-sql">&lt;span style="color: Green">RECEIVES_OCCURING&lt;/span></code>: the activated procedure is issuing <code class="prettyprint lang-sql">&lt;span style="color: Blue">RECEIVE&lt;/span></code> command against the monitored queue.

The important thing to notice here is that the Queue Monitor will go into <code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code> state when an activated procedure was launched and will not launch again any procedure until a <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statement is issued against the monitored queue. This is done in order to prevent situations when a queue is incorrectly configured and launches the wrong stored procedure into execution. Altough in practice is the activated procedure that is expected to execute the <code class="prettyprint lang-sql">&lt;span style="color: Blue">RECEIVE&lt;/span></code> statement, this is not enforced by the Queue Monitor. Any connection that executes a <code class="prettyprint lang-sql">&lt;span style="color: Blue">RECEIVE&lt;/span></code> while the Queue Monitor is in <code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code> state will cause the Queue Monitor to enter the <code class="prettyprint lang-sql">&lt;span style="color: Green">RECEIVES_OCCURING&lt;/span></code> state.

This activation state machine also handles sending the <code class="prettyprint lang-sql">QUEUE_ACTIVATION</code> events for <a href="http://msdn.microsoft.com/en-us/library/ms171581.aspx" target="_blank">Event-Based Activation</a>. The only difference that instead of launching a stored procedure into execution the Queue Monitor sends a message of type <code class="prettyprint lang-sql">&lt;span style="color:Green">http://schemas.microsoft.com/SQL/Notifications/EventNotification&lt;/span></code> to the notification subscriber. Again, same as with internal activation, once the notification is sent the Queue Monitor enters the <code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code> state and will not send more notifications messages until a <code class="prettyprint lang-sql">&lt;span style="color: Blue">RECEIVE&lt;/span></code> statement is executed on the monitored queue.

The Queue Monitor will go back to <code class="prettyprint lang-sql">&lt;span style="color: Green">INACTIVE&lt;/span></code> state when there are no more messages ready to be received in the queue.

## Queue Monitor Timers

The Queue Monitors are also responsible for handling the activation ramp up for handling a higher load by launching more instances of the activated procedure. Once the Queue Monitor enters the <code class="prettyprint lang-sql">&lt;span style="color: Green">RECEIVES_OCCURING&lt;/span></code> state (ie. a <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statement is issued) it will start an internal timer. Each time a <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statement on the monitored queue returns an empty rowset this timer is reset. If this timer ever reaches a threshold then a new instance of the activated procedure is launched. The reasoning behind this mechanism is that activated procedures are supposed to issue <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statements and process the received messages. If <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> does not find any available messages and returns an empty rowset the activation timer is reset because it means that there are no messages available for receive, so there is no reason to launch a new instance of the procedure. The timer threshold is 5 seconds and currently is hardcoded.

This timers are also used for sending new <code class="prettyprint lang-sql">QUEUE_ACTIVATION</code> event messages if the Event-Based Activation is used instead of internal activation.

## Troubleshooting

If you find yourself looking at Queue Monitors in <code class="prettyprint lang-sql">&lt;span style="color: Green">sys.dm_broker_queue_monitors&lt;/span></code> is usualy because of one reason: your activated procedure is failing to launch.

If there is no record for your queue in <code class="prettyprint lang-sql">&lt;span style="color: Green">sys.dm_broker_queue_monitors&lt;/span></code> then this indicates that the Service Broker activation is not even monitoring the queue. For activation to monitor the queue several settings have to be properly configured:



  * The queue has to be enabled. Check <code class="prettyprint lang-sql">is_receive_enabled</code> and <code class="prettyprint lang-sql">is enqueue_enabled</code> in <code class="prettyprint lang-sql">&lt;span style="color: green">sys.service_queues&lt;/span></code>.
  * Activation has to be enabled on the queue. Check <code class="prettyprint lang-sql">is_activation_enabled</code> in <code class="prettyprint lang-sql">&lt;span style="color: green">sys.service_queues&lt;/span></code>.
  * Activation has to be properly configured. Check <code class="prettyprint lang-sql">activation_procedure</code>, <code class="prettyprint lang-sql">max_queue_readers</code> and <code class="prettyprint lang-sql">execute_ss_principal_id</code> in <code class="prettyprint lang-sql">&lt;span style="color: green">sys.service_queues&lt;/span></code>.
  * The Service Broker in the database has to be enabled. Check <code class="prettyprint lang-sql">is_broker_enabled</code> in <code class="prettyprint lang-sql">&lt;span style="color: green">sys.databases&lt;/span></code>.



If there is a record for your queue in <code class="prettyprint lang-sql">&lt;span style="color: Green">sys.dm_broker_queue_monitors&lt;/span></code> and its state is <code class="prettyprint lang-sql">&lt;span style="color: Green">INACTIVE&lt;/span></code> then there are no messages in the queue ready to be received. Check the content of the queue with a <code class="prettyprint lang-sql">&lt;span style="color:Blue">SELECT&lt;/span>&nbsp;*&nbsp;&lt;span style="color:Blue">FROM&lt;/span>&nbsp;&lt;span style="color:Green">[&lt;your_queue_name&gt;]&lt;/span></code>. Messages ready to be received have to be unlocked and the status has to be 1 <code class="prettyprint lang-sql">&lt;span style="color:Green">ready&lt;/span></code>. Note that the BOL documentation of <a href="http://msdn.microsoft.com/en-us/library/ms186963.aspx" target="_blank">RECEIVE</a> erroneously shows the state 0 as <code class="prettyprint lang-sql">&lt;span style="color:Green">ready&lt;/span></code>, which is incorrect.

If the Queue Monitor for your queue is in <a name="hint"></a><code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code> and it stays in that state is an indication that your activated procedure is not issuing the correct <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statement. Check your activated procedure. Note that if you change the procedure the Queue Monitor will **not** detect this change and launch again the procedure. The easiest thing to do is to recycle the Queue Monitor by disabling and then enabling activation on the queue:  
<code class="prettyprint lang-sql">&lt;br />
&lt;span style="color: Blue">ALTER QUEUE&lt;/span> [&lt;span style="color: Green">&lt;your_queue&gt;&lt;/span>] &lt;span style="color: Blue">WITH ACTIVATION (STATUS = OFF)&lt;/span>;&lt;br />
&lt;span style="color: Blue">ALTER QUEUE&lt;/span> [&lt;span style="color: Green">&lt;your_queue&gt;&lt;/span>] &lt;span style="color: Blue">WITH ACTIVATION (STATUS = ON)&lt;/span>;</code> 

Finally if the Queue Monitor for your queue is in the state <span style="color: Green">RECEIVES_OCCURING</span> then it indicates that the <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> statement is being issued against the queue. Make sure your assesment that the procedure is not being launched is correct. Check if there are other applications or procedures that are erroneously receiving from your queue.

## Demo Caution

I will end with a cautionary tale. I was at a presentation where the speaker was showing a very nice monitoring facility he had built around the event notifications for DDL that he was using to monitor a farm of SQL Server instances. As he was making his way through the demo, he showed how it attaches a stored procedure to the queue to monitor for the notification messages. But in order to not perturb the presentation flow (at moment focused on setting up activation) he did _not_ create the whole procedure, just a stub for it. Something like this:  
<code class="prettyprint lang-sql">&lt;br />
&lt;span style="color: Blue">CREATE PROCEDURE&lt;/span>&nbsp;usp_NotificationProcessor&lt;br />
&lt;span style="color: Blue">AS&lt;br />
BEGIN&lt;/span>&lt;br />
	&lt;span style="color:Green">--TODO: add code here&lt;/span>&lt;br />
&lt;span style="color: Blue">END&lt;/span>;&lt;br />
</code> 

The speaker then continued with his demo and attached the procedure to the queue for activation. Now since he had already created the subscription for all DDL events on the demo database, his queue was receiving the notification messaged as he was creating and altering objects in the database.

He finaly reaches the point when he changes the stub procedure to actually process the notifications and then the audience eagerly awaits the results. And nothing happens.

Can you guess what was wrong? Go back and read [this](#hint) for a hint.

Since activation was enabled and there were messages to received, the Queue Monitor has launched the procedure altough it was nothing but a stub. The Queue Monitor has entered the <code class="prettyprint lang-sql">&lt;span style="color: Green">NOTIFIED&lt;/span></code> state and would not launch again the procedure until a <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code> was issued. Later when the stub procedure was changed to do the actual processing, the procedure would not be launched by the Queue Monitor because it would still wait for <code class="prettyprint lang-sql">&lt;span style="color:Blue">RECEIVE&lt;/span></code>. This can be easily fixed by turning activation on the queue off and back on, as I showed before.