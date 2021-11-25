---
id: 26
title: 'Fire and Forget: Good for the military, but not for Service Broker conversations'
date: 2006-04-06T10:14:39+00:00
author: remus
layout: post
guid: http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/
permalink: /2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/
categories:
  - Troubleshooting
---
<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span><span>  </span>@dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Initiator]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> N<span style="color: red">&#8216;Target&#8217;</span><span style="color: gray">;</span><span style="color: red"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SEND</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Does the code above look familiar? Well, it shouldn’t, because it contains a very nasty error. This is the dreaded fire-and-forget message exchange pattern. The initiator opens a conversation, sends a message and then ends the conversation. I often seen this pattern deployed in practice, and (just apparently!) it seems to works fine. The target receives the message sent, and it ends its own target endpoint of the conversation, and things look <a href="http://en.wikipedia.org/wiki/Ren_and_Stimpy">happy, happy, happy, joy, joy, joy</a>. Then suddenly something changes, and messages seem to vanish in thin air. The initiator sends them for sure, but the target doesn’t get them. The sent message is not in sys.transmission_queue, is not in the target queue, there is no error in the initiator queue, and even more, the initiator endpoint disappears! What happened? Well, the target service has responded with an error to the message the initiator has sent. The error could happen for a multitude of reasons, most common being one of these:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>the target may had denied access to the initiator service
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>the message XML validation has failed (the target may had added a schema validation to the message)
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>the target service may no longer implement the contract used by the initiator
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>the target service was just dropped (can happen only if the broker instance was specified in the BEGIN DIALOG)
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Well, if the target responded with an error, then why isn’t the error enqueued in the initiator’s queue? Because the initiator has declared that it is no interested in seeing that error! It had ended the conversation! So when the error comes back from the target, the error message is dropped and the initiator endpoint is deleted. This is the expected behavior when the conversation endpoint is in DISCONNECTED_OUTBOUND state.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The solution is let the target end the conversation first. <span> </span>The initiator can simply send the message to the target and then continue. The target will receive the message, end the conversation from the target side and this will send an EndDialog message back to the initiator. All we have to do is attach an activated stored procedure to the initiator’s queue, and in this stored procedure we can end the initiator side of the conversation:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> inititator_queue_activated_procedure<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @message_type <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @message_body <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">WAITFOR</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">RECEIVE</span> @dh <span style="color: gray">=</span> [conversation_handle]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>@message_type <span style="color: gray">=</span> [message_type_name]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>@message_body <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>[message_body] <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">))</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">FROM</span> [InitiatorQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">WHILE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: gray">IS</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">IF</span> @message_type <span style="color: gray">=</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Received error %s from service [Target]&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @message_body<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">SELECT</span> @dh <span style="color: gray">=</span> <span style="color: gray">NULL;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">RECEIVE</span> @dh <span style="color: gray">=</span> [conversation_handle]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span>@message_type <span style="color: gray">=</span> [message_type_name]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span>@message_body <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>[message_body] <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">))</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">FROM</span> [InitiatorQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">COMMIT</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [InitiatorQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WITH</span> ACTIVATION <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>STATUS <span style="color: gray">=</span> <span style="color: blue">ON</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>MAX_QUEUE_READERS <span style="color: gray">=</span> 1<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>PROCEDURE_NAME <span style="color: gray">=</span> [inititator_queue_activated_procedure]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">EXECUTE</span> <span style="color: blue">AS</span> OWNER<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Now when en error is returned by the target service it is logged into the System Event Log:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Event Type:<span>   </span>Information<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Event Source:<span> </span>MSSQLSERVER<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Event Category:<span>      </span>(2)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Event ID:<span>     </span>17061<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Date:<span>         </span>4/6/2006<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Time:<span>         </span>10:01:41 PM<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">User:<span>         </span>REDMOND\remusr<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Computer:<span>     </span>REMUSR10<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Description:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'">Error: 50000 Severity: 10 State: 1 Received error ?<?xml version=&#8221;1.0&#8243;?><Error xmlns=&#8221;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8221;><Code>-8425</Code><Description>The service contract &apos;DEFAULT&apos; is not found.</Description></Error> from service [Target]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 9pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  We could have chosen to insert the error message into an audit table, or even send it to an audit service (using Service Broker, of course.<span style="font-family: Wingdings"><span>J</span></span>).
</p>