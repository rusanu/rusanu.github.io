---
id: 33
title: Recycling Conversations
date: 2007-05-03T09:32:00+00:00
author: remus
layout: post
guid: /2007/05/03/recycling-conversations/
permalink: /2007/05/03/recycling-conversations/
categories:
  - CodeProject
  - Samples
  - Tutorials
tags:
  - Code
  - conversation
  - performance
  - sample
  - service broker
  - sql
  - tsql
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">In my previous post<font color="#800080"> <a href="/2007/04/25/reusing-conversations/">Reusing Conversations</a></font></font><a href="http://blogs.msdn.com/remusrusanu/archive/2007/04/24/reusing-conversations.aspx"></a><font size="3"> I promised I’ll follow up with a solution to the question about how to end the conversations that are reused for the data-push scenario (logging, auditing, ETL for DW etc).</font></span><!--more-->
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
  <span style="font-family: Verdana"><font size="3">So how should we end this dialogs? My recommendation is to end them based on time. Choose a time for how long a dialog should be used, depending on the expected number of messages traffic. Then you can use the Service Broker </font><a href="http://msdn2.microsoft.com/en-us/library/ms187804.aspx"><font color="#800080" size="3">conversation timers</font></a><font size="3"> to end the dialog when it’s time is done. I also believe that the timer should be handled by the initiator, but the conversation should be ended by the target. Given that the pattern we’re talking about (one way data push) involves sending only messages from initiator to target, the initiator is not allowed to end the conversation because that will create a fire-and-forget message pattern, a very undesirable message exchange pattern, see </font><a href="/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/"><font color="#800080" size="3">/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/</font></a><font size="3">. My recommendation is to have a special message in the contract used by the two services that informs that target that the initiator is done sending on this dialog. Lets call this message [EndOfStream]. The target would respond to this message by ending its side of the conversation, thus sending an EndDialog message to the initiator. The initiator responds to this message by ending its side (issuing the END CONVERSATION verb).<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-family: Verdana"><font size="3">To implement what I described above we would need to modify the original example from </font><a href="/2007/04/25/reusing-conversations/"><font color="#800080" size="3">/2007/04/25/reusing-conversations/</font></a><font size="3"> to create a conversation timer when the dialog is created and to add an activated procedure that handles the timer message, as well as the EndDialog and Error messages. One interesting issue is how to handle the timer message. After sending the [EndOfStream] message, the initiator should not send any more messages on that dialog. This is easily achieved by removing the dialog handle from the </font></span><font size="3"><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table. The next invocation of usp_Send will create a new one, just as desired. But there is a race condition there, an usp_Send procedure might be executing already and trying to send a message on that conversation. Even though the dialog was removed from the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table, the concurrent usp_Send will successfully send on the dialog that just sent an [EndOfStream] message. Since the target responds to the [EndOfstream] message by ending the conversation, the message sent is most likely lost as the target will ignore it! To solve this situation, the usp_Send procedure must guarantee the stability of the conversation it looked up in the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table. So the SELECT from the </span><span style="font-family: 'Courier New'">[SessionConversations] </span><span style="font-family: Verdana">table has to be REPEATABLE, and this can be achieved with an appropriate query hint (<a href="http://msdn2.microsoft.com/en-us/library/ms187373.aspx"><font color="#800080">HOLDLOCK</font></a>)<o:p></o:p></span></font>. <b>Update:</b> Several readers have deployed this into production and found out that HOLDLOCK is not enough and UPDLOCK behaves better. The code is updated to reflect this.
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

<pre><span style="color: Black"></span><span style="color:Green">--&nbsp;This&nbsp;table&nbsp;associates&nbsp;the&nbsp;current&nbsp;connection&nbsp;(@@SPID)<br />
--&nbsp;with&nbsp;a&nbsp;conversation.&nbsp;The&nbsp;association&nbsp;key&nbsp;also<br />
--&nbsp;contains&nbsp;the&nbsp;conversation&nbsp;parameteres:<br />
--&nbsp;from&nbsp;service,&nbsp;to&nbsp;service,&nbsp;contract&nbsp;used<br />
--<br />
</span><span style="color:Blue">CREATE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TABLE</span><span style="color:Black">&nbsp;[SessionConversations]&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">	SPID&nbsp;</span><span style="color:Blue">INT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL</span><span style="color:Black">	<br />
	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;FromService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ToService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;OnContract&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;Handle&nbsp;</span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Blue">PRIMARY</span><span style="color:Black">&nbsp;</span><span style="color:Blue">KEY</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;FromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ToService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;OnContract</span><span style="color:Gray">)<br />
</span><span style="color:Black">	</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Blue">UNIQUE</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">Handle</span><span style="color:Gray">));<br />
</span><span style="color:Black">GO<br />
<br />
</span><span style="color:Green">--&nbsp;SEND&nbsp;procedure.&nbsp;Will&nbsp;lookup&nbsp;to&nbsp;reuse&nbsp;an&nbsp;existing&nbsp;conversation<br />
--&nbsp;or&nbsp;start&nbsp;a&nbsp;new&nbsp;in&nbsp;case&nbsp;no&nbsp;conversation&nbsp;exists&nbsp;or&nbsp;the&nbsp;conversation<br />
--&nbsp;cannot&nbsp;be&nbsp;used<br />
--<br />
</span><span style="color:Blue">CREATE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">PROCEDURE</span><span style="color:Black">&nbsp;[usp_Send]&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">@fromService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@toService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@onContract&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@messageType&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@messageBody&nbsp;</span><span style="color:Blue">VARBINARY</span><span style="color:Gray">(</span><span style="color:Fuchsia">MAX</span><span style="color:Gray">))<br />
</span><span style="color:Blue">AS<br />
BEGIN<br />
</span><span style="color:Black">	</span><span style="color:Blue">SET</span><span style="color:Black">&nbsp;</span><span style="color:Blue">NOCOUNT</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Blue">INT</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Blue">INT</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TRANSACTION</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Green">--&nbsp;Will&nbsp;need&nbsp;a&nbsp;loop&nbsp;to&nbsp;retry&nbsp;in&nbsp;case&nbsp;the&nbsp;conversation&nbsp;is<br />
</span><span style="color:Black">	</span><span style="color:Green">--&nbsp;in&nbsp;a&nbsp;state&nbsp;that&nbsp;does&nbsp;not&nbsp;allow&nbsp;transmission<br />
</span><span style="color:Black">	</span><span style="color:Green">--<br />
</span><span style="color:Black">	</span><span style="color:Blue">WHILE</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Seek&nbsp;an&nbsp;eligible&nbsp;conversation&nbsp;in&nbsp;[SessionConversations]<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;Handle<br />
			</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[SessionConversations]&nbsp;</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Blue">UPDLOCK</span><span style="color:Gray">)<br />
</span><span style="color:Black">			</span><span style="color:Blue">WHERE</span><span style="color:Black">&nbsp;SPID&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@SPID<br />
</span><span style="color:Black">			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;FromService&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@fromService<br />
			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;ToService&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@toService<br />
			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;OnContract&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@OnContract</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">IS</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Need&nbsp;to&nbsp;start&nbsp;a&nbsp;new&nbsp;conversation&nbsp;for&nbsp;the&nbsp;current&nbsp;@@spid<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">DIALOG</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle<br />
				</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;</span><span style="color:Blue">SERVICE</span><span style="color:Black">&nbsp;@fromService<br />
				</span><span style="color:Blue">TO</span><span style="color:Black">&nbsp;</span><span style="color:Blue">SERVICE</span><span style="color:Black">&nbsp;@toService<br />
				</span><span style="color:Blue">ON</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONTRACT</span><span style="color:Black">&nbsp;@onContract<br />
				</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ENCRYPTION</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">OFF</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Set&nbsp;an&nbsp;one&nbsp;hour&nbsp;timer&nbsp;on&nbsp;the&nbsp;conversation<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;TIMER&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">@handle</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TIMEOUT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;3600</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">INSERT</span><span style="color:Black">&nbsp;</span><span style="color:Blue">INTO</span><span style="color:Black">&nbsp;[SessionConversations]<br />
				</span><span style="color:Gray">(</span><span style="color:Black">SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;FromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ToService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;OnContract</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;Handle</span><span style="color:Gray">)<br />
</span><span style="color:Black">				</span><span style="color:Blue">VALUES<br />
</span><span style="color:Black">				</span><span style="color:Gray">(</span><span style="color:Fuchsia">@@SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@fromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@toService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@onContract</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">);<br />
</span><span style="color:Black">		</span><span style="color:Blue">END</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Attempt&nbsp;to&nbsp;SEND&nbsp;on&nbsp;the&nbsp;associated&nbsp;conversation<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">SEND</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle<br />
			</span><span style="color:Blue">MESSAGE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TYPE</span><span style="color:Black">&nbsp;@messageType<br />
			</span><span style="color:Gray">(</span><span style="color:Black">@messageBody</span><span style="color:Gray">);<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@ERROR</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Successful&nbsp;send,&nbsp;just&nbsp;exit&nbsp;the&nbsp;loop<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">BREAK</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@counter</span><span style="color:Gray">+</span><span style="color:Black">1</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">&gt;</span><span style="color:Black">&nbsp;10<br />
		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;We&nbsp;failed&nbsp;10&nbsp;times&nbsp;in&nbsp;a&nbsp;row,&nbsp;something&nbsp;must&nbsp;be&nbsp;broken<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">RAISERROR</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">				N</span><span style="color:Red">'Failed&nbsp;to&nbsp;SEND&nbsp;on&nbsp;a&nbsp;conversation&nbsp;for&nbsp;more&nbsp;than&nbsp;10&nbsp;times.&nbsp;Error&nbsp;%i.'<br />
</span><span style="color:Black">				</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;16</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@error</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">LOG</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">BREAK</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Delete&nbsp;the&nbsp;associated&nbsp;conversation&nbsp;from&nbsp;the&nbsp;table&nbsp;and&nbsp;try&nbsp;again<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">DELETE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[SessionConversations]<br />
			</span><span style="color:Blue">WHERE</span><span style="color:Black">&nbsp;Handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL;<br />
</span><span style="color:Black">	</span><span style="color:Blue">END<br />
</span><span style="color:Black">	</span><span style="color:Blue">COMMIT</span><span style="color:Gray">;<br />
</span><span style="color:Blue">END<br />
</span><span style="color:Black">GO<br />
<br />
<br />
</span><span style="color:Blue">CREATE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">PROCEDURE</span><span style="color:Black">&nbsp;usp_SenderActivation<br />
</span><span style="color:Blue">AS<br />
BEGIN<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@messageTypeName&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@messageBody&nbsp;</span><span style="color:Blue">VARBINARY</span><span style="color:Gray">(</span><span style="color:Fuchsia">MAX</span><span style="color:Gray">);<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TRANSACTION</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">RECEIVE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TOP</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Black">		@handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">conversation_handle</span><span style="color:Gray">,<br />
</span><span style="color:Black">		@messageTypeName&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;message_type_name</span><span style="color:Gray">,<br />
</span><span style="color:Black">		@messageBody&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;message_body<br />
		</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[sender_queue]</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">IS</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Delete&nbsp;the&nbsp;message&nbsp;from&nbsp;the&nbsp;[SessionConversations]&nbsp;table<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;before&nbsp;sending&nbsp;the&nbsp;[EndOfStream]&nbsp;message.&nbsp;The&nbsp;order&nbsp;is<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;important&nbsp;to&nbsp;avoid&nbsp;deadlocks.&nbsp;Strictly&nbsp;speaking,&nbsp;we&nbsp;should<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;only&nbsp;delete&nbsp;if&nbsp;the&nbsp;message&nbsp;type&nbsp;is&nbsp;timer&nbsp;or&nbsp;error,&nbsp;but&nbsp;is<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;simpler&nbsp;and&nbsp;safer&nbsp;to&nbsp;just&nbsp;delete&nbsp;always<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">DELETE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[SessionConversations]<br />
			</span><span style="color:Blue">WHERE</span><span style="color:Black">&nbsp;[Handle]&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@messageTypeName&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/DialogTimer'<br />
</span><span style="color:Black">		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Blue">SEND</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle&nbsp;<br />
				</span><span style="color:Blue">MESSAGE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TYPE</span><span style="color:Black">&nbsp;[EndOfStream]</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Blue">ELSE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@messageTypeName&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'<br />
</span><span style="color:Black">		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Blue">END</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Blue">ELSE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@messageTypeName&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'<br />
</span><span style="color:Black">		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Blue">END</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Insert&nbsp;your&nbsp;error&nbsp;handling&nbsp;here.&nbsp;Could&nbsp;send&nbsp;a&nbsp;notification&nbsp;or<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;store&nbsp;the&nbsp;error&nbsp;in&nbsp;a&nbsp;table&nbsp;for&nbsp;further&nbsp;inspection<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;We’re&nbsp;gonna&nbsp;log&nbsp;the&nbsp;error&nbsp;into&nbsp;the&nbsp;ERRORLOG&nbsp;and&nbsp;Eventvwr.exe<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Blue">INT</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@description&nbsp;</span><span style="color:Blue">NVARCHAR</span><span style="color:Gray">(</span><span style="color:Black">4000</span><span style="color:Gray">);<br />
</span><span style="color:Black">			</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Blue">XMLNAMESPACES</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'</span><span style="color:Black">&nbsp;</span><span style="color:Blue">AS</span><span style="color:Black">&nbsp;ssb</span><span style="color:Gray">)<br />
</span><span style="color:Black">			</span><span style="color:Blue">SELECT<br />
</span><span style="color:Black">				@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">CAST</span><span style="color:Gray">(</span><span style="color:Black">@messageBody&nbsp;</span><span style="color:Blue">AS</span><span style="color:Black">&nbsp;</span><span style="color:Blue">XML</span><span style="color:Gray">).</span><span style="color:Black">value</span><span style="color:Gray">(<br />
</span><span style="color:Black">					</span><span style="color:Red">'(//ssb:Error/ssb:Code)[1]'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Red">'INT'</span><span style="color:Gray">),<br />
</span><span style="color:Black">				@description&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">CAST</span><span style="color:Gray">(</span><span style="color:Black">@messageBody&nbsp;</span><span style="color:Blue">AS</span><span style="color:Black">&nbsp;</span><span style="color:Blue">XML</span><span style="color:Gray">).</span><span style="color:Black">value</span><span style="color:Gray">(<br />
</span><span style="color:Black">					</span><span style="color:Red">'(//ssb:Error/ssb:Description)[1]'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Red">'NVARCHAR(4000)'</span><span style="color:Gray">)<br />
</span><span style="color:Black">			</span><span style="color:Blue">RAISERROR</span><span style="color:Gray">(</span><span style="color:Black">N</span><span style="color:Red">'Received&nbsp;error&nbsp;Code:%i&nbsp;Description:"%s"'</span><span style="color:Gray">,<br />
</span><span style="color:Black">				&nbsp;16</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@error</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@description</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">LOG</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">END<br />
</span><span style="color:Black">	</span><span style="color:Blue">COMMIT</span><span style="color:Gray">;<br />
</span><span style="color:Blue">END<br />
</span><span style="color:Black">GO<br />
<br />
</span><span style="color:Green">--&nbsp;attach&nbsp;the&nbsp;uspSenderActivation&nbsp;procedure&nbsp;with&nbsp;the&nbsp;sender’s&nbsp;queue<br />
--<br />
</span><span style="color:Blue">ALTER</span><span style="color:Black">&nbsp;</span><span style="color:Blue">QUEUE</span><span style="color:Black">&nbsp;[sender_queue]<br />
	</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;ACTIVATION&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">	</span><span style="color:Blue">STATUS</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Gray">,<br />
</span><span style="color:Black">	MAX_QUEUE_READERS&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,<br />
</span><span style="color:Black">	PROCEDURE_NAME&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;[usp_SenderActivation]</span><span style="color:Gray">,<br />
</span><span style="color:Black">	</span><span style="color:Blue">EXECUTE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">AS</span><span style="color:Black">&nbsp;</span><span style="color:Blue">OWNER</span><span style="color:Gray">);<br />
</span><span style="color:Black">GO</span>
</pre>