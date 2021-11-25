---
id: 173
title: Parallel Activation
date: 2007-10-29T12:53:00+00:00
author: remus
layout: revision
guid: http://rusanu.com/2007/10/29/29-revision/
permalink: /2007/10/29/29-revision/
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">A frequent question from customers is ‘How can I change the activation timing?’. The short answer is that you cannot, this is hard coded in the SQL Server engine and cannot be customized. But the reason why most people want the timing changed is to start all the configured max_queue_readers at once. I will show you an unorthodox trick that can be used to achieve this result.</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <!--more-->
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Normally the activation mechanism monitors the queues and the RECEIVEs occurring and decides when is appropriate to launch a new instance of the activated procedure. However there is another, less known, side of activation: the </font><a href="http://msdn2.microsoft.com/en-us/library/ms189453.aspx"><font face="Times New Roman" size="3">QUEUE_ACTIVATION</font></a><font face="Times New Roman" size="3"> event notification. Those of you familiar with the </font><a href="http://www.gotdotnet.com/codegallery/codegallery.aspx?id=9f7ae2af-31aa-44dd-9ee8-6b6b6d3d6319"><font face="Times New Roman" size="3">External Activation sample</font></a><font face="Times New Roman" size="3"> for sure have learned about these events. This works similarly to the classical activated procedure, but instead of launching an instance of the activated procedure, a notification is sent on the subscribed service. The key difference is that there is no restrictions on how many different notification subscriptions can be created for the same QUEUE_ACTIVATION event! And when there is the time to activate, <em>all</em> the subscribers are notified. These notifications are ultimately ordinary Service Broker messages sent to the subscribed service. These subscribed services are running on queues that, obviously, can have attached procedures to be activated. See where I am going? You can use the subscribed service’s queue activation to launch a separate procedure <em>per subscribed service</em> for each original queue activation notification. So if you create 5 QUEUE_ACTIVATION subscriptions from 5 separate services, you will launch 5 procedures (nearly) simultaneously!</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Here is a code example showing this:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; This is the real application queue that receives the application messages<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'" lang="FR">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ApplicationQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">service</span> [ApplicationService] <span style="color: blue">on</span> <span style="color: blue">queue</span> [ApplicationQueue] <span style="color: gray">(</span>[DEFAULT]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">go<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; These are the activation queues that receive the event notifications<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'" lang="FR">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ActivatorQueue_1]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ActivatorQueue_2]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ActivatorQueue_3]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ActivatorQueue_4]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'" lang="FR"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">service</span> [ActivatorService_1] <span style="color: blue">on</span> <span style="color: blue">queue</span> [ActivatorQueue_1] <span style="color: gray">(</span>[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">service</span> [ActivatorService_2] <span style="color: blue">on</span> <span style="color: blue">queue</span> [ActivatorQueue_2] <span style="color: gray">(</span>[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">service</span> [ActivatorService_3] <span style="color: blue">on</span> <span style="color: blue">queue</span> [ActivatorQueue_3] <span style="color: gray">(</span>[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">service</span> [ActivatorService_4] <span style="color: blue">on</span> <span style="color: blue">queue</span> [ActivatorQueue_4] <span style="color: gray">(</span>[http://schemas.microsoft.com/SQL/Notifications/PostEventNotification]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">go<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Now create the activated procedures. Note that they will be activated on the [ActivatorQeueu_&#8230; queues, but they have to RECEIVE from the original [ApplicationQueue]. But they cannot simply ignore the [ActivatorQueue_&#8230;], they have to RECEIVE the message that activated them, because otherwise the activation on [ActivatorQueue_&#8230;] will simply stop activating them! </font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">procedure</span> [ApplicationActivationHelper<span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">_1]</span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">as<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">begin<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">set</span> <span style="color: blue">nocount</span> <span style="color: blue">on</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">declare</span> @conversationHandle <span style="color: blue">uniqueidentifier</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">declare</span> @messageTypeName <span style="color: blue">sysname</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">declare</span> @notification <span style="color: blue">xml</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">declare</span> @applicationMessage <span style="color: blue">varbinary</span><span style="color: gray">(</span><span style="color: fuchsia">max</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">declare</span> @error <span style="color: blue">nvarchar</span><span style="color: gray">(</span><span style="color: fuchsia">max</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin</span> <span style="color: blue">transaction</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RECEIVE</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@conversationHandle <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageTypeName <span style="color: gray">=</span> message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@notification <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">as</span> <span style="color: blue">XML</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [ActivatorQueue<span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">_1]<span style="color: gray">;</span></span><span style="color: gray"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">if</span> <span style="color: gray">(</span>@messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; This is an error received on the notification dialog<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; log it to the event log<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">select</span> @error <span style="color: gray">=</span> <span style="color: fuchsia">cast</span><span style="color: gray">(</span>@notification <span style="color: blue">as</span> <span style="color: blue">nvarchar</span><span style="color: gray">(</span><span style="color: fuchsia">max</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">raiserror</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Notification activator [ActivatorQueue<span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">_1]:</span> Received error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @error<span style="color: gray">)</span> <span style="color: blue">with</span> <span style="color: fuchsia">log</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end</span> <span style="color: blue">conversation</span> @conversationHandle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">else</span> <span style="color: blue">if</span> <span style="color: gray">(</span>@messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end</span> <span style="color: blue">conversation</span> @conversationHandle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Done consumming the event notification, if any, continue<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; with the actual application logic<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span> <span style="color: blue">RECEIVE</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span>@conversationHandle <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageTypeName <span style="color: gray">=</span> message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@applicationMessage <span style="color: gray">=</span> message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [ApplicationQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">if</span> <span style="color: gray">(</span><span style="color: fuchsia">@@rowcount</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; do NOT rollback here<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">break</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">if</span> <span style="color: gray">(</span>@messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; This is an error received on the application dialog<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; log it to the event log<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">select</span> @error <span style="color: gray">=</span> <span style="color: fuchsia">cast</span><span style="color: gray">(</span>@notification <span style="color: blue">as</span> <span style="color: blue">nvarchar</span><span style="color: gray">(</span><span style="color: fuchsia">max</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">raiserror</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Application: Received error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @error<span style="color: gray">)</span> <span style="color: blue">with</span> <span style="color: fuchsia">log</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end</span> <span style="color: blue">conversation</span> @conversationHandle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">else</span> <span style="color: blue">if</span> <span style="color: gray">(</span>@messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end</span> <span style="color: blue">conversation</span> @conversationHandle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">else</span> <span style="color: blue">if</span> <span style="color: gray">(</span>@messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">begin<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Here goes the real application logic.<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; In our example, simply send back an echo<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">send</span> <span style="color: blue">on</span> <span style="color: blue">conversation</span> @conversationHandle <span style="color: gray">(</span>@applicationMessage<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">end<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">commit</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">end<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">go<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">alter</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">queue</span> [ActivatorQueue<span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">_1]</span> <span style="color: blue">with</span> activation <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>status <span style="color: gray">=</span> <span style="color: blue">on</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>max_queue_readers <span style="color: gray">=</span> 1<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>procedure_name <span style="color: gray">=</span> [ApplicationActivationHelper<span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">_1]</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">execute</span> <span style="color: blue">as</span> owner<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font size="3"><font face="Times New Roman">You’re gonna have to repeat this 3 more times, for each [ActivatorQueue_&#8230;]. I highlighted the places where you need to change the </font><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'">_1</span><font face="Times New Roman"> into </font><span style="font-family: 'Courier New'">_2, _3, _4</span><font face="Times New Roman"> etc. BTW, what happens if we change the number of max_queue_readers here? We’ll get a ‘batch’ behavior, where whenever the [ApplicationQueue] monitoring believes there is need to activate one more reader, a whole <em>set</em> of readers will be activated, up to the max_queue_readers * number of subscribed services.</font></font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Now create the 4 event notification subscriptions:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Now create the event notifications, one for each activator service<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">event</span> <span style="color: blue">notification</span> [ActivatorEvent_1]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">on</span> <span style="color: blue">queue</span> [ApplicationQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">for</span> QUEUE_ACTIVATION<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">to</span> <span style="color: blue">service</span> N<span style="color: red">&#8216;ActivatorService_1&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;current database&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">event</span> <span style="color: blue">notification</span> [ActivatorEvent_2]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span></span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">on</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">queue</span> [ApplicationQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"><span> </span><span style="color: blue">for</span> QUEUE_ACTIVATION<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"><span> </span><span style="color: blue">to</span> <span style="color: blue">service</span> N<span style="color: red">&#8216;ActivatorService_2&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;current database&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'" lang="FR">create</span><span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"> <span style="color: blue">event</span> <span style="color: blue">notification</span> [ActivatorEvent_3]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"><span> </span><span style="color: blue">on</span> <span style="color: blue">queue</span> [ApplicationQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"><span> </span><span style="color: blue">for</span> QUEUE_ACTIVATION<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'" lang="FR"><span> </span><span style="color: blue">to</span> <span style="color: blue">service</span> N<span style="color: red">&#8216;ActivatorService_3&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;current database&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">event</span> <span style="color: blue">notification</span> [ActivatorEvent_4]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">on</span> <span style="color: blue">queue</span> [ApplicationQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">for</span> QUEUE_ACTIVATION<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">to</span> <span style="color: blue">service</span> N<span style="color: red">&#8216;ActivatorService_4&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;current database&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">go<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">Notice how the 4 subscriptions are for the same event (QUEUE_ACTIVATION on [ApplicationQueue]), but the subscribed service is different for each one.</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font size="3"><font face="Times New Roman">Last step we need to do is verify how all fits together. Attach the Profiler to the server and monitor the Broker/Broker:Activation event to see how the procedures are activated/shut down. Then we can send one message to the </font><span style="font-family: 'Courier New'">ApplicationService</span><font face="Times New Roman"> and see how 4 activated procedures will race at once to grab this message. Of course, only one will succeed in our example, the other 3 will time out and deactivate themselves:</font></font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">queue</span> [Initiator]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">create</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">service</span> [Initiator] <span style="color: blue">on</span> <span style="color: blue">queue</span> [Initiator]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">go<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">declare</span><span style="font-size: 10pt; font-family: 'Courier New'"> @h <span style="color: blue">uniqueidentifier</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">begin</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">dialog</span> <span style="color: blue">conversation</span> @h<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">from</span> <span style="color: blue">service</span> [Initiator]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">to</span> <span style="color: blue">service</span> N<span style="color: red">&#8216;ApplicationService&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;current database&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">with</span> encryption <span style="color: gray">=</span> <span style="color: blue">off</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">send</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">on</span> <span style="color: blue">conversation</span> @h <span style="color: gray">(</span>N<span style="color: red">&#8216;Test&#8217;</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font face="Times New Roman" size="3">The question arises now: is this for real? Should a production system use this trick? I’d recommend against. The interaction between the QUEUE_ACTIVATION event on the [ApplicationQueue] and the internal activation on the [ActivatorQueue_&#8230;] queues is complex and very easy to get awry. The maintenance of the almost identical [</font><span style="font-size: 10pt; font-family: 'Courier New'">ApplicationActivationHelper_&#8230;] </span><font size="3"><font face="Times New Roman">procedures is difficult, any change has to me replicated to all the procedures. Changing the number of activated procedures is difficult and cumbersome; it involves creating and dropping new event notification, as well as the entire necessary infrastructure to each subscribed service: queue, service and procedure. However, if you really believe you need to activate multiple procedures at once, this trick can help you. For sure it can be impressive for demonstrations </font><span style="font-family: Wingdings"><span>J</span></span></font>
</p>

<img src="http://blogs.msdn.com/aggbug.aspx?PostID=890877" height="1" width="1" />