---
id: 459
title: Reusing Conversations
date: 2009-05-17T04:16:52+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/05/17/32-revision-3/
permalink: /2009/05/17/32-revision-3/
---
<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana">One of the most common deployed patterns of using Service Broker is what I would call &#8216;data push&#8217;, when Service Broker conversations are used to send data one way only (from initiator to target). In this pattern the target never sends any message back to the initiator. Common applications for this pattern are:<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">ETL from the transactions system to the data warehouse<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">audit and logging<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">aggregation of data from multiple sites<o:p></o:p></span>
</p>

<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana"><o:p> </o:p></span>
</p>

<p style="text-indent: 0in">
  <!--more-->
  
  <span style="font-size: 10pt; color: black; font-family: Verdana">Implementing this pattern is usually very trivial: modify a stored procedure to SEND a message (a datagram) or add a trigger on a table to issue the SEND. But as soon as this is done, the next question that arises is: do I start a new conversation for each message or do I try to reuse conversations? My answer would be that reusing conversations for such patterns (one way data push) is highly recommended.<o:p></o:p></span>
</p>

<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana">Here are some of my arguments for reusing conversations:<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">conversations are the only elements that <em>are guaranteed</em> to preserve order in transmission (even locally!). E.g. if the order of the events audited is important, they have to be sent on the same conversation<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">reusing conversations has a significant performance impact (see <a href="http://blogs.msdn.com/remusrusanu/archive/2007/04/03/orlando-slides-and-code.aspx"><font color="#800080">http://blogs.msdn.com/remusrusanu/archive/2007/04/03/orlando-slides-and-code.aspx</font></a>)<o:p></o:p></span>
</p>

<p style="margin-left: 1in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: 'Courier New'"><span>o<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">the cost of setting up and tearing down a conversation for each message can influence the performance by ~4x <o:p></o:p></span>
</p>

<p style="margin-left: 1in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: 'Courier New'"><span>o<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">the advantage on having multiple messages on the same conversation can speed up processing on the RECEIVE side by a factor of ~10x<o:p></o:p></span>
</p>

<ul style="margin-top: 0in" type="disc">
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">conversations are the &#8216;unit of work&#8217; for the Service Broker background and as such resources are allocated in the system according to the number of active conversations (&#8216;active&#8217; = have some activity pending, like sending a message or replying with an acknowledgement). The number of pending messages has less influence on resource allocation (2 dialogs with 500 messages each will allocate far less resources than 1000 dialogs with 1 message each)<o:p></o:p></span>
  </li>
</ul>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">The next step is to decide just how many conversations to use. One conversation would completely preserve order of operations, but because each SEND locks the conversation exclusively, would serialize every transaction and lead to serious scalability problems. My favorite scheme is to have one conversation <em>per <span> </span>user connection</em>. This would preserve the order of actions on one connection, which is most often the desired granularity, and will scale up easily because is very unusual for connections to share transactions (just how often is it that you use <a href="http://msdn2.microsoft.com/en-us/library/ms180061.aspx"><font color="#800080">sp_getbindtoken</font></a>?).<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">Next is an example stored proc that reuses conversations based on the conversation arguments (from service, to service, contract) and current connection ID (@@SPID). It&#8217;s based on a table to store the conversation associated with each @@SPID.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; This table associates the current connection (@@SPID) <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; with a conversation. The association key also<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; contains the conversation parameteres:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; from service, to service, contract used<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TABLE</span> [SessionConversations] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>SPID <span style="color: blue">INT</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>FromService <span style="color: blue">SYSNAME</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>ToService <span style="color: blue">SYSNAME</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>OnContract <span style="color: blue">SYSNAME</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>Handle <span style="color: blue">UNIQUEIDENTIFIER</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">PRIMARY</span> <span style="color: blue">KEY</span> <span style="color: gray">(</span>SPID<span style="color: gray">,</span> FromService<span style="color: gray">,</span> ToService<span style="color: gray">,</span> OnContract<span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">UNIQUE</span> <span style="color: gray">(</span>Handle<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; SEND procedure. Will lookup to reuse an existing conversation<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; or start a new in case no conversation exists or the conversation<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; cannot be used<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [usp_Send] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@fromService <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@toService <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@onContract <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageType <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageBody <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">))<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SET</span> <span style="color: blue">NOCOUNT</span> <span style="color: blue">ON</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @counter <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @counter <span style="color: gray">=</span> 1<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Will need a loop to retry in case the conversation is <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; in a state that does not allow transmission<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Seek an eligible conversation in [SessionConversations]<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @handle <span style="color: gray">=</span> Handle<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionConversations]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> SPID <span style="color: gray">=</span> <span style="color: fuchsia">@@SPID<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">AND</span> FromService <span style="color: gray">=</span> @fromService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: gray">AND</span> ToService <span style="color: gray">=</span> @toService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">AND</span> OnContract <span style="color: gray">=</span> @OnContract<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @handle <span style="color: gray">IS</span> <span style="color: gray">NULL<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Need to start a new conversation for the current @@spid <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @handle<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> @fromService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> @toService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> @onContract<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INSERT</span> <span style="color: blue">INTO</span> [SessionConversations] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>SPID<span style="color: gray">,</span> FromService<span style="color: gray">,</span> ToService<span style="color: gray">,</span> OnContract<span style="color: gray">,</span> Handle<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">VALUES<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span><span style="color: fuchsia">@@SPID</span><span style="color: gray">,</span> @fromService<span style="color: gray">,</span> @toService<span style="color: gray">,</span> @onContract<span style="color: gray">,</span> @handle<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Attempt to SEND on the associated conversation<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @handle<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> @messageType<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>@messageBody<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @error <span style="color: gray">=</span> <span style="color: fuchsia">@@ERROR</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @error <span style="color: gray">=</span> 0<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Successful send, just exit the loop<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BREAK</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @counter <span style="color: gray">=</span> @counter<span style="color: gray">+</span>1<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @counter <span style="color: gray">></span> 10<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; We failed 10 times in a<span> </span>row, something must be broken<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Failed to SEND on a conversation for more than 10 times. Error %i.&#8217;</span><span style="color: gray">,</span> 16<span style="color: gray">,</span> 1<span style="color: gray">,</span> @error<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BREAK</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Delete the associated conversation from the table and try again<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> [SessionConversations]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> Handle <span style="color: gray">=</span> @handle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @handle <span style="color: gray">=</span> <span style="color: gray">NULL;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Test the procedure with some dummy paramaters. The FromService &#8216;ST&#8217;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; must already exist in the database<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">exec</span><span style="font-size: 10pt; font-family: 'Courier New'"> usp_Send <span style="color: red">&#8216;ST&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;Foobar&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span><span style="color: gray">,</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span><span style="color: gray">,</span> 0x01020304<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">This shows how to reuse the conversations, but there are still plenty of details one has to think of, like:<o:p></o:p></span>
</p>

<ul style="margin-top: 0in" type="disc">
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">how long should a conversation be used for?<o:p></o:p></span>
  </li>
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">who should end these reused conversations?<o:p></o:p></span>
  </li>
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">how to treat conversation errors?<o:p></o:p></span>
  </li>
</ul>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">I&#8217;ll try to visit these topics in a future post.</span>
</p>

<img src="http://blogs.msdn.com/aggbug.aspx?PostID=2265845" height="1" width="1" />