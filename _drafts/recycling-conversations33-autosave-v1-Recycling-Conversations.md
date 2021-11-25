---
id: 455
title: Recycling Conversations
date: 2009-07-21T18:00:57+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/07/09/33-autosave/
permalink: /2009/07/21/33-autosave-v1/
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">In my previous post<font color="#800080"> <a href="http://rusanu.com/2007/04/25/reusing-conversations/">Reusing Conversations</a></font></font><a href="http://blogs.msdn.com/remusrusanu/archive/2007/04/24/reusing-conversations.aspx"></a><font size="3"> I promised I’ll follow up with a solution to the question about how to end the conversations that are reused for the data-push scenario (logging, auditing, ETL for DW etc).</font></span><!--more-->
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">First of all, why end them in the first place? The answer is to minimize exposure in case of problems. If a problem happens and a dialog has to be destroyed (i.e. errored), then we want to minimize the potential number of messages that we have to re-process or that are potential lost. But wait a minute, how come we’re talking about message loss, doesn’t Service Broker guarantee delivery? The truth is that no system can ever guarantee delivery <em>under all conditions</em>. Human error can happen resulting in misconfiguration, applications have bugs that can send invalid messages (e.g. fail XML validation); hardware can be catastrophically lost w/o any possibility to restore a database/host and so on and so forth. Reusing a single dialog to send all your messages (or one per @@spid) is like putting all your eggs in one basket. If that dialog was trailing millions of messages behind, all those messages have to be reprocessed or are lost. <o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">Another reason to recycle the dialogs is to allow RETENTION on queues. Enabling RETENTION keeps a copy of every message sent to/from that queue <em>for the duration of the dialog</em>. If a dialog is kept open for unlimited duration, the queue will continuously grow.<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">Also we want to recycle dialogs to deal with load balancing scenarios, since a dialog is sticky to one service instance we would like to periodically start new dialogs so they ‘stick’ to new instances of from the available target service instances.<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">So how should we end this dialogs? My recommendation is to end them based on time. Choose a time for how long a dialog should be used, depending on the expected number of messages traffic. Then you can use the Service Broker </font><a href="http://msdn2.microsoft.com/en-us/library/ms187804.aspx"><font color="#800080" size="3">conversation timers</font></a><font size="3"> to end the dialog when it’s time is done. I also believe that the timer should be handled by the initiator, but the conversation should be ended by the target. Given that the pattern we’re talking about (one way data push) involves sending only messages from initiator to target, the initiator is not allowed to end the conversation because that will create a fire-and-forget message pattern, a very undesirable message exchange pattern, see </font><a href="hhttp://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/"><font color="#800080" size="3">http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/</font></a><font size="3">. My recommendation is to have a special message in the contract used by the two services that informs that target that the initiator is done sending on this dialog. Lets call this message [EndOfStream]. The target would respond to this message by ending its side of the conversation, thus sending an EndDialog message to the initiator. The initiator responds to this message by ending its side (issuing the END CONVERSATION verb).<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">To implement what I described above we would need to modify the original example from </font><a href="http://rusanu.com/2007/04/25/reusing-conversations/"><font color="#800080" size="3">http://rusanu.com/2007/04/25/reusing-conversations/</font></a><font size="3"> to create a conversation timer when the dialog is created and to add an activated procedure that handles the timer message, as well as the EndDialog and Error messages. One interesting issue is how to handle the timer message. After sending the [EndOfStream] message, the initiator should not send any more messages on that dialog. This is easily achieved by removing the dialog handle from the </font></span><font size="3"><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table. The next invocation of usp_Send will create a new one, just as desired. But there is a race condition there, an usp_Send procedure might be executing already and trying to send a message on that conversation. Even though the dialog was removed from the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table, the concurrent usp_Send will successfully send on the dialog that just sent an [EndOfStream] message. Since the target responds to the [EndOfstream] message by ending the conversation, the message sent is most likely lost as the target will ignore it! To solve this situation, the usp_Send procedure must guarantee the stability of the conversation it looked up in the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table. So the SELECT from the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table has to be REPEATABLE, and this can be achieved with an appropriate query hint (<a href="http://msdn2.microsoft.com/en-us/library/ms187373.aspx"><font color="#800080">HOLDLOCK</font></a>)<o:p></o:p></span></font>. <b>Update:</b> Several readers have deployed this into production and found out that HOLDLOCK is not enough and UPDLOCK behaves better. The code is updated to reflect this.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font size="3"><span style="font-family: Verdana">But by doing this we introduce a new problem: the activated procedure handling the timer message and the usp_Send procedure might deadlock each other! One will hold a lock on the conversation and try to update the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table, while the other will hold the lock on the </span><span style="font-family: 'Courier New'">[SessionConversations]</span><span style="font-family: Verdana"> table and try to acquire the lock on the conversation: guaranteed deadlock. To solve this problem, will solve it as most deadlock problems are solved: acquire the locks in the <em>same</em> order in both procedures. Since the usp_Send must acquire the locks in the order row-in-</span><span style="font-family: 'Courier New'">[SessionConversations]-</span><span style="font-family: Verdana">table-first-conversation-next, the activated procedure must do the same. That is: delete the row from </span><span style="font-family: 'Courier New'">[SessionConversations]</span><span style="font-family: Verdana"> first, send the [EndOfStream] message next.<o:p></o:p></span></font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><o:p><font size="3"> </font></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">So here is the code, the modified usp_Send and the initiator side activated procedure.<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><o:p><font size="3"> </font></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><o:p><font size="3"> </font></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionConversations] <span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">WITH</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> <span style="color: gray">(</span><span style="color: blue">UPDLOCK</span><span style="color: gray">)</span></span><span style="color: gray"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> SPID <span style="color: gray">=</span> <span style="color: fuchsia">@@SPID<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">AND</span> FromService <span style="color: gray">=</span> @fromService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">AND</span> ToService <span style="color: gray">=</span> @toService<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: gray">AND</span> OnContract <span style="color: gray">=</span> @OnContract<span style="color: gray">;<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> @onContract<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> <span style="color: blue">ENCRYPTION</span> <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: green">&#8212; Set an one hour timer on the conversation<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">CONVERSATION</span> TIMER <span style="color: gray">(</span>@handle<span style="color: gray">)</span> <span style="color: blue">TIMEOUT</span> <span style="color: gray">=</span> 3600<span style="color: gray">;</span></span><span style="font-size: 10pt; font-family: 'Courier New'"> <o:p></o:p></span>
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
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> usp_SenderActivation<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @messageTypeName <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @messageBody <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span><span style="color: gray">(</span>1<span style="color: gray">)</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@handle <span style="color: gray">=</span> <span style="color: blue">conversation_handle</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageTypeName <span style="color: gray">=</span> message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageBody <span style="color: gray">=</span> message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [sender_queue]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @handle <span style="color: gray">IS</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Delete the message from the [SessionConversations] table<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; before sending the [EndOfStream] message. The order is <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; important to avoid deadlocks. Strictly speaking, we should<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; only delete if the message type is timer or error, but is<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; simpler and safer to just delete always<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> [SessionConversations]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> [Handle] <span style="color: gray">=</span> @handle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/DialogTimer&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @handle <span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [EndOfStream]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> @messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @handle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> @messageTypeName <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @handle<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Insert your error handling here. Could send a notification or<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; store the error in a table for further inspection <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; We&#8217;re gonna log the error into the ERRORLOG and Eventvwr.exe<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @description <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> <span style="color: blue">XMLNAMESPACES</span> <span style="color: gray">(</span><span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span> <span style="color: blue">AS</span> ssb<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@error <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@messageBody <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">).</span>value<span style="color: gray">(</span><span style="color: red">&#8216;(//ssb:Error/ssb:Code)[1]&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;INT&#8217;</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@description <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@messageBody <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">).</span>value<span style="color: gray">(</span><span style="color: red">&#8216;(//ssb:Error/ssb:Description)[1]&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;NVARCHAR(4000)&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span><span style="color: gray">(</span>N<span style="color: red">&#8216;Received error Code:%i Description:&#8221;%s&#8221;&#8217;</span><span style="color: gray">,</span> 16<span style="color: gray">,</span> 1<span style="color: gray">,</span> @error<span style="color: gray">,</span> @description<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">COMMIT</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">;<o:p></o:p></span>
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
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; attach the uspSenderActivation procedure with the sender&#8217;s queue<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [sender_queue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ACTIVATION <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">STATUS</span> <span style="color: gray">=</span> <span style="color: blue">ON</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>MAX_QUEUE_READERS <span style="color: gray">=</span> 1<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>PROCEDURE_NAME <span style="color: gray">=</span> [usp_SenderActivation]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">EXECUTE</span> <span style="color: blue">AS</span> <span style="color: blue">OWNER</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO</span>
</p>

<img src="http://blogs.msdn.com/aggbug.aspx?PostID=2388436" height="1" width="1" />