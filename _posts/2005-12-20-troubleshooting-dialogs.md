---
id: 21
title: Troubleshooting Dialogs
date: 2005-12-20T09:14:04+00:00
author: remus
layout: post
guid: http://rusanu.com/2005/12/20/troubleshooting-dialogs/
permalink: /2005/12/20/troubleshooting-dialogs/
categories:
  - Troubleshooting
tags:
  - error
  - service broker
  - Troubleshooting
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So you built your first Service Broker app, you’ve sent the first message and now you’re looking for the message on the target queue. Yet, the message is not there. What do you do? Where do you look first? Well, troubleshooting Service Broker is a bit different than troubleshooting your everyday database app.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So I’m trying to build here a short guide that you can follow to troubleshoot Service Broker issues.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  First, you should ensure that the message was actually sent and committed.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Next, check if the message exists in the <span style="font-family: 'Courier New'">sys.transmission_queue</span>. The transmission queue is similar to an Outgoing mailbox, an &#8216;Outbox&#8217;. Messages are kept there until the target acknowledges that it received the message, after which they are deleted. Therefore, if the message is still in the transmission queue it means that it was not yet acknowledged by the destination. How does one diagnose what is the problem? My recommendation is to follow the message flow: message is sent by sender, then accepted by the target, then an ack is sent by the target and finally this ack is accepted by the sender. I any of these steps have a problem, then the message will sit in the <span style="font-family: 'Courier New'">sys.transmission_queue</span>.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Let&#8217;s now look at how to diagnose each step of the message flow. BTW, I often refer to the acknowledgement as &#8216;ack&#8217;, and to the transmission queue as &#8216;xmit queue&#8217;.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  1<strong>. The sender cannot send the message, for whatever reason</strong>. If this is the case, the <span style="font-family: 'Courier New'">transmission_status</span> column in <span style="font-family: 'Courier New'">sys.transmission_queue </span>will contain an error message that will point at the problem. The appropriate action depends on the error being displayed.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Common problems include security problems (no database master key, no remote service binding, no certificates etc), classification problem (no route for the target service etc) or adjacent transport connection issues (connection handshake errors, unreachable target host etc)
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  2. <strong>The sender does send the message and the message reaches the target but the target does not accept the message. </strong>In this case, the sender&#8217;s <span style="font-family: 'Courier New'">transmission_status</span> will be empty. To diagnose this issue, you must attach the Profiler to the target machine and enable the following events: <span style="color: black; font-family: 'Courier New'">&#8216;Broker/</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">Broker:Conversation&#8217;, &#8216;Broker/Broker:Message Undeliverable&#8217; and &#8216;Broker/Broker:Remote Message Acknowledgement&#8217;. </span><span style="color: black">When the message arrives, you will see the event &#8216;</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">Broker/Broker:Remote Message Acknowledgement</span><span style="color: black">&#8216; with the EventSubClass </span><span style="font-size: 10pt; color: black">&#8216;</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">Message with Acknowledgement Received</span><span style="font-size: 10pt; color: black">&#8216;</span><span style="color: black"> followed by &#8216;</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">Broker/Broker:Message Undeliverable</span><span style="color: black">&#8216; event. The TextData of this last event will contain an error message that will point at the problem.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="color: black">Common problem in this case are security problems (you must turn on in addition the &#8216;</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">Audit Security/Audit Broker Conversation</span><span style="color: black">&#8216; event in the Profiler to investigate these problems, the TextData should pinpoint to the failure cause), typos in service names or broker instance id, disabled target queues.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="color: black">Note that in case this error in TextData says </span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">&#8216;This message could not be delivered because it is a duplicate.&#8217; </span><span style="color: black">it means that the message is actually accepted by the target, but the acks don&#8217;t reach back to the sender and therefore the sender is retrying the message again and again (see below).</span><span style="color: black; font-family: 'Courier New'"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  3. <strong>The sender does send the message, the message reaches the target and is accepted, but the target is unable to send back an ack</strong>. Same as above, you must attach the Profiler to the target machine and you will see repeated occurrences of the &#8216;<span style="font-size: 10pt; color: black; font-family: 'Courier New'">Broker/Broker:Message Undeliverable</span><span style="color: black">&#8216;</span> event with the TextData &#8216;<span style="font-size: 10pt; color: black; font-family: 'Courier New'">This message could not be delivered because it is a duplicate.</span>&#8216;. The vent will be generated each time the sender is retrying the message, which happens about once/minute (strictly speaking is after 4, 8, 16, 32 and then once every 64 seconds).
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Typically the problem is a misconfigured route back from the target to the sender (the route for the initiator service is missing). The Profiler event &#8216;<span style="font-size: 10pt; color: black; font-family: 'Courier New'">Broker:Message Classify</span><span style="color: black">&#8216; will show this, with an EventSubClass &#8216;</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">3 &#8211; Delayed&#8217; </span><span style="color: black">and a TextData message of </span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">&#8216;The target service name could not be found. Ensure that the service name is specified correctly and/or the routing information has been supplied.&#8217;.</span><span style="color: black"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Another possible cause is when the route configured on the target for the sender service has a typo. Since the ack is not stored in the sys.transmission_queue, you don&#8217;t have the handy transmission_status. Or do you? Actually, you can use the <span style="font-family: 'Courier New'">get_transmission_status</span> function to get the transmission status of the ack! Lookup the conversation handle is sys.conversation_endpoints and then use this function to query the transmission status of the ack sent by that dialog back to the sender.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  4. <strong>The sender does send the message, but the message never reaches the target. </strong>This can happen only if there are intermediate hops (SQL Server instances acting as forwarders). To determine which forwarder drops the messages, you have to connect the Profiler to each forwarder and see which one traces <span style="font-size: 10pt; color: black; font-family: 'Courier New'">&#8216;Broker:Forwarded Message Dropped&#8217; </span><span style="color: black">events. The most likely causes are either message timeout (the forwarders can&#8217;t get to send the message in time due to high load) or a misconfigured routing information on the forwarder (like missing routes in MSDB database, which is the one used for forwarding).<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  5. <strong>The sender does send the message, the message reaches the target and is accepted, the target is sending back the ack but the ack never reaches back the initiator </strong>(again, a forwarder is required for this to happen). Investigating this issue is identical with the issue above: attach the Profiler to each forwarder until you find the one that is dropping messages. Note that the forwarders from the sender to the target are not necessarily the same as the ones on the route from the target back to the sender!
</p>