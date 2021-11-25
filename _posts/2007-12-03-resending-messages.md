---
id: 71
title: Resending messages
date: 2007-12-03T20:29:15+00:00
author: remus
layout: post
guid: http://rusanu.com/2007/12/03/resending-messages/
permalink: /2007/12/03/resending-messages/
categories:
  - CodeProject
  - Samples
  - Tutorials
tags:
  - performance
  - sample
  - service broker
  - sql
  - tsql
  - tutorial
---
<p __designer:dtid="281474976710660" style="margin: 0in 0in 0pt" class="MsoNormal">This article is a continuation of my two articles on using Service Broker as a pure data-push one-way communication channel: <a __designer:dtid="281474976710662" href="http://rusanu.com/2007/04/25/reusing-conversations/">Reusing Conversations</a> and <a __designer:dtid="281474976710663" href="http://rusanu.com/2007/05/03/recycling-conversations/">Recycling Conversations</a>. I originally did not plan for this third part, but I was more than once asked the same question: if a conversation is in error what happens to the messages that were sent but not yet delivered?</p> <p _\_designer:dtid="281474976710664">The short answer is that messages that are still pending are in the sender’s database sys.transmission\_queue system table so when an Error message is received the sender can scan this table and resend each message that is still pending. An example of such procedure is not difficult to code:</p> 

<!--more-->

<pre><code class="prettyprint lang-sql">
-- Resends all pending messages in sys.transmission_queue
-- belonging to an old colversation on a new conversation
--
CREATE PROCEDURE usp_ResendPending (
      @oldConversation UNIQUEIDENTIFIER,
      @newConversation UNIQUEIDENTIFIER)
AS
BEGIN
      DECLARE @mt SYSNAME;
      DECLARE @body VARBINARY;
 
      -- Must declare a cursor to iterate over
      -- all the pending messages
      -- It is important to keep the message order
      -- and to keep the original message type
      DECLARE cursorPending CURSOR
            LOCAL
            FORWARD_ONLY
            READ_ONLY
            FOR SELECT [message_type_name],
                        [message_body]
                  FROM sys.[transmission_queue]
                  WHERE [conversation_handle] = @oldConversation
                  ORDER BY message_sequence_number;
      OPEN cursorPending;
     
      FETCH NEXT FROM cursorPending
            INTO @mt, @body;
      WHILE (@@FETCH_STATUS = 0)
      BEGIN
            -- Resend the message on the new conversation
            IF (@body IS NOT NULL)
            BEGIN
                  -- If the @body is not null it must be sent explicitly
                  SEND ON CONVERSATION (@newConversation)
                        MESSAGE TYPE @mt
                        (@body);
            END
            ELSE
            BEGIN
                  -- Messages with no body must *not* specify the body,
                  -- cannot send a NULL value argument
                  --
                  SEND ON CONVERSATION (@newConversation)
                        MESSAGE TYPE @mt;
            END
            FETCH NEXT FROM cursorPending
                  INTO @mt, @body;
      END
     
      CLOSE cursorPending;
      DEALLOCATE cursorPending;
END
GO
</code></pre>

Now the interesting question here is not how to do this but <em __designer:dtid="281474976710966">when</em> to do this. My previous article had this code that was dealing with error messages:

<pre><code class="prettyprint lang-sql">
ELSE IF @messageTypeName = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
BEGIN
      -- Insert your error handling here. Could send a notification or
      -- store the error in a table for further inspection
      -- We’re gonna log the error into the ERRORLOG and Eventvwr.exe
      --
      DECLARE @error INT;
      DECLARE @description NVARCHAR(4000);
      WITH XMLNAMESPACES ('http://schemas.microsoft.com/SQL/ServiceBroker/Error' AS ssb)
      SELECT
            @error = CAST(@messageBody AS XML).value(
					'(//ssb:Error/ssb:Code)[1]', 'INT'),
            @description = CAST(@messageBody AS XML).value(
					'(//ssb:Error/ssb:Description)[1]', 'NVARCHAR(4000)');
      RAISERROR(N'Received error Code:%i Description:"%s"', 
			16, 1, @error, @description) WITH LOG;

</code><code style="background:yellow">
      -- Insert call to usp_ResendPending here</code><code class="prettyprint lang-sql">
           
      -- END CONVERSATION has to be issued AFTER the call to usp_ResendPending
      END CONVERSATION @handle;
END
</code></pre><p __designer:dtid="281474976711107">I’ve already highlighted the place where the call to the resending procedure should be inserted. <strong __designer:dtid="281474976711108">Important note</strong>: the place where the END CONVERSATION is issued in code has moved from my earlier post because once a conversation is ended all pending messages in sys.transmission_queue are deleted. So now your sender is going to resend any pending message in case it receives an error. But what sort of errors may the sender receive? Now that is no longer a trivial problem.</p> <h2 __designer:dtid="281474976711109" style="margin: 12pt 0in 3pt"><em __designer:dtid="281474976711110"><span __designer:dtid="281474976711111" style="font-size: 14pt; font-family: Arial">Service Broker conversation errors</span></em></h2> <p _\_designer:dtid="281474976711112">The message\_body of a conversation error message contains the error code and error description. There are two types of errors one conversation can receive:</p> <p __designer:dtid="281474976711113" style="margin-left: 0.5in; text-indent: -0.25in; tab-stops: list .5in"><span __designer:dtid="281474976711114" style="font-family: Symbol"><span __designer:dtid="281474976711115">·<span __designer:dtid="281474976711116" style="font: 7pt 'Times New Roman'">         </span></span></span><strong __designer:dtid="281474976711117">application errors</strong> that were sent by the other endpoint of the conversation using the END CONVERSATION … WITH ERROR … verb. The error code for these errors is a positive number.</p> <p __designer:dtid="281474976711118" style="margin-left: 0.5in; text-indent: -0.25in; tab-stops: list .5in"><span __designer:dtid="281474976711119" style="font-family: Symbol"><span __designer:dtid="281474976711120">·<span __designer:dtid="281474976711121" style="font: 7pt 'Times New Roman'">         </span></span></span><strong __designer:dtid="281474976711122">system errors</strong> that were sent by the Service Broker infrastructure when an error condition has occurred. The error code for these errors is a negative number and its value is negated value of a SQL Server error code from <span __designer:dtid="281474976711123" style="font-family: 'Courier New'">sys.messages</span>.</p> <p __designer:dtid="281474976711124">I cannot comment anything about the application error values because, of course, they are application specific and is probably you that defined them in the first place. But I can give details on the system error codes what they mean:</p> <p __designer:dtid="281474976711125" style="margin-top: 6pt"><span __designer:dtid="281474976711126" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711127">8494<span __designer:dtid="281474976711128">     </span>You do not have permission to access the service &#8216;%.*ls&#8217;.</strong></span><br __designer:dtid="281474976711129" />This system error is sent to a conversation that is trying to reach a service that is denying access to the sender.</p> <p __designer:dtid="281474976711130" style="margin-top: 6pt"><span __designer:dtid="281474976711131" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711132">8489<span __designer:dtid="281474976711133">     </span>The dialog has exceeded the specified LIFETIME.<br __designer:dtid="281474976711134" /></strong></span>This system error is sent to a conversation when the lifetime specified during BEGIN DIALOG has expired. Note that all conversations have a lifetime, if you omitted it then the lifetime is the 2<sup __designer:dtid="281474976711135">31</sup> seconds (that is roughly 68 years)</p> <p __designer:dtid="281474976711136" style="margin-top: 6pt"><span __designer:dtid="281474976711137" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711138">8462<span __designer:dtid="281474976711139">     </span>The remote conversation endpoint is either in a state where no more messages can be exchanged, or it has been dropped.<br __designer:dtid="281474976711140" /></strong></span>This system error is sent on a conversation that continued to send messages to a peer that has already closed it’s endpoint.</p> <p __designer:dtid="281474976711141" style="margin-top: 6pt"><span __designer:dtid="281474976711142" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711143">8470<span __designer:dtid="281474976711144">     </span>Remote service has been dropped.<br __designer:dtid="281474976711145" /></strong></span>This<span __designer:dtid="281474976711146">  </span>system error is sent on each active conversation that was initiated from or targeted a service that was DROPed from the database</p> <p __designer:dtid="281474976711147" style="margin-top: 6pt"><span __designer:dtid="281474976711148" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711149">8469<span __designer:dtid="281474976711150">     </span>Remote service has been altered.</strong></span><strong __designer:dtid="281474976711151"><br __designer:dtid="281474976711152" /></strong>This<span __designer:dtid="281474976711153">  </span>system error is sent on each active conversation that was initiated from or targeted a service that was ALTERed</p> <p __designer:dtid="281474976711154" style="margin-top: 6pt"><span __designer:dtid="281474976711155" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711156">8487<span __designer:dtid="281474976711157">     </span>The remote service contract has been dropped<br __designer:dtid="281474976711158" /></strong></span>This system error is sent on each active conversation that was initiated using the contract that is being DROPed from the database</p> <p __designer:dtid="281474976711159" style="margin-top: 6pt"><span __designer:dtid="281474976711160" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711161">8490<span __designer:dtid="281474976711162">     </span>Cannot find the remote service &#8216;%.*ls&#8217; because it does not exist.</strong></span><strong __designer:dtid="281474976711163"><br __designer:dtid="281474976711164" /></strong>This system error is sent to a conversation that is trying to reach a service in a specific broker instance and the service does not exist. This can happen if the optional broker instance is specified during BEGIN DIALOG</p> <p __designer:dtid="281474976711165" style="margin-top: 6pt"><span __designer:dtid="281474976711166" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711167">8425<span __designer:dtid="281474976711168">     </span>The service contract &#8216;%.*ls&#8217; is not found.</strong></span><strong __designer:dtid="281474976711169"><br __designer:dtid="281474976711170" /></strong>This system error is sent to a conversation that is using a contract not installed on the peer database</p> <p __designer:dtid="281474976711171" style="margin-top: 6pt"><span __designer:dtid="281474976711172" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711173">8408<span __designer:dtid="281474976711174">     </span>Target service &#8216;%.\*ls&#8217; does not support contract &#8216;%.\*ls&#8217;.<br __designer:dtid="281474976711175" /></strong></span>This system error is sent to a conversation that is using a contract not bound to the target service</p> <p __designer:dtid="281474976711176" style="margin-top: 6pt"><span __designer:dtid="281474976711177" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711178">8428<span __designer:dtid="281474976711179">     </span>The message type &#8220;%.*ls&#8221; is not found.<br __designer:dtid="281474976711180" /></strong></span>This system error is sent to a conversation that is using a message type not found in the target database</p> <p __designer:dtid="281474976711181" style="margin-top: 6pt"><span __designer:dtid="281474976711182" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711183">8498<span __designer:dtid="281474976711184">     </span>The remote service has sent a message of type &#8216;%.*ls&#8217; that is not part of the local contract.<br __designer:dtid="281474976711185" /></strong></span>This system error is sent to a conversation that is using a message not bound to the contract used</p> <p __designer:dtid="281474976711186" style="margin-top: 6pt"><span __designer:dtid="281474976711187" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711188">8430<span __designer:dtid="281474976711189">     </span>The message body failed the configured validation.<br __designer:dtid="281474976711190" /></strong></span>This system error is sent to a conversation that is using a message type validation (<span _\_designer:dtid="281474976711191" style="font-family: 'Courier New'">none, empty, well\_formed_xml</span>) that does not agree with the peer validation for the same message type.</p> <p __designer:dtid="281474976711192" style="margin-top: 6pt"><span __designer:dtid="281474976711193" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711194">8457<span __designer:dtid="281474976711195">     </span>The message received was sent by the initiator of the conversation, but the message type &#8216;%.*ls&#8217; is marked SENT BY TARGET in the contract.<br __designer:dtid="281474976711196" /></strong></span>This system error is sent to a conversation that has sent a message from initiator but the message is marked as SENT BY TARGET in the contract</p> <p __designer:dtid="281474976711197" style="margin-top: 6pt"><span __designer:dtid="281474976711198" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711199">8437<span __designer:dtid="281474976711200">     </span>The message received was sent by a Target service, but the message type &#8216;%.*ls&#8217; is marked SENT BY INITIATOR in the contract.<br __designer:dtid="281474976711201" /></strong></span>This system error is sent to a conversation that has sent a message from target but the message is marked as SENT BY INITIATOR in the contract</p> <p __designer:dtid="281474976711202" style="margin-top: 6pt"><span __designer:dtid="281474976711203" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711204">8495<span __designer:dtid="281474976711205">     </span>The conversation has already been acknowledged by another instance of this service.<br __designer:dtid="281474976711206" /></strong></span>This system error is sent to a conversation that is sending a reply to an initiator but the initiator is rejecting this peer because has already established conversation with another peer. This scenario can happen only in certain conditions when the load-balancing routes of Service Broker are used.</p> <p __designer:dtid="281474976711207" style="margin-top: 6pt"><span __designer:dtid="281474976711208" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711209">9616<span __designer:dtid="281474976711210">     </span>A message of type &#8216;%.*ls&#8217; was received and failed XML validation.<span __designer:dtid="281474976711211">  </span>%.\*ls This occurred in the message with Conversation ID &#8216;%.\*ls&#8217;, Initiator: %d, and Message sequence number: %I64d.<br __designer:dtid="281474976711212" /></strong></span>This system error is sent to a conversation that has sent a message type marked as conforming to a certain XML schema but the payload has failed to pass the XML validation for the said schema</p> <p __designer:dtid="281474976711213" style="margin-top: 6pt"><span __designer:dtid="281474976711214" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711215">9719<span __designer:dtid="281474976711216">     </span>The database for this conversation endpoint is attached or restored.<br __designer:dtid="281474976711217" /></strong></span>This system error is sent to any active conversation when the ALTER DATABASE … SET ERROR\_BROKER\_CONVERSATIONS is issued. The text of the error message comes from the fact that this ALTER statement is usually issued in conjunction with restore or attach operations.</p> <p __designer:dtid="281474976711218" style="margin-top: 6pt"><span __designer:dtid="281474976711219" style="font-family: 'Courier New'">&#8211;<strong __designer:dtid="281474976711220">28052<span __designer:dtid="281474976711221">    </span>Cannot decrypt session key while regenerating master key with FORCE option.<br __designer:dtid="281474976711222" /></strong></span>This system error is sent to a conversation when the database master key was regenerated with FORCE and the conversation sessions keys (encrypted with the database master key) are lost</p> <p __designer:dtid="281474976711225">Although the list of system errors is quite large (and I actually left out three more errors that I really think too much of a corner case to worth mention) we can distinguish three cases:</p> <p __designer:dtid="281474976711226" style="margin-left: 0.5in; text-indent: -0.25in; tab-stops: list .5in"><span __designer:dtid="281474976711227">1.<span __designer:dtid="281474976711228" style="font: 7pt 'Times New Roman'">      </span></span>Errors that we can recover from immediately and can continue sending messages, including resend any pending messages. These are 8489 LIFETIME expired, 8462 Peer has closed, 9719 attach/restore and 28052 Master key regenerate.</p> <p __designer:dtid="281474976711229" style="margin-left: 0.5in; text-indent: -0.25in; tab-stops: list .5in"><span __designer:dtid="281474976711230">2.<span __designer:dtid="281474976711231" style="font: 7pt 'Times New Roman'">      </span></span>Errors that are the result of a breach of contract between the two parties (initiator and target). There is no point in retrying in this case since these problems indicate either an application defect or a deployment configuration problem. The list of these errors is 8494 Access denied, 8490 Service does not exist and 8425, 8408, 8428, 8498, 8430, 8457, 8437, 9616 all of which are a form of service contract agreement violation. In case you wonder how can one hit errors like 8437 and why isn’t SQL Server itself enforcing the correct SENT BY TARGET/SENT BY INITIATOR when the SEND verb is issued this is because the message type and contract definition may be different between the initiator and target database.</p> <p __designer:dtid="281474976711232" style="margin-left: 0.5in; text-indent: -0.25in; tab-stops: list .5in"><span __designer:dtid="281474976711233">3.<span __designer:dtid="281474976711234" style="font: 7pt 'Times New Roman'">      </span></span>Error that are the result of a database DROP or ALTER operation and the retry result is undefined. These are 8470, 8487 and 8469.</p> <p __designer:dtid="281474976711237">The appropriate action for each case depends on you application semantics. For my example of reusing/recycling dialogs I would only take the action of resending pending messages for errors in the first case 8489, 8462, 9719 and 28052:</p> 

<pre><code class="prettyprint lang-sql">
...
ELSE IF @messageTypeName = N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'
BEGIN
      -- Insert your error handling here. Could send a notification or
      -- store the error in a table for further inspection
      -- We’re gonna log the error into the ERRORLOG and Eventvwr.exe
      --
      DECLARE @error INT;
      DECLARE @description NVARCHAR(4000);
      WITH XMLNAMESPACES ('http://schemas.microsoft.com/SQL/ServiceBroker/Error' AS ssb)
      SELECT
            @error = CAST(@messageBody AS XML).value(
					'(//ssb:Error/ssb:Code)[1]', 'INT'),
            @description = CAST(@messageBody AS XML).value(
					'(//ssb:Error/ssb:Description)[1]', 'NVARCHAR(4000)');
      RAISERROR(N'Received error Code:%i Description:"%s"', 
			16, 1, @error, @description) WITH LOG;
     
      IF (@error IN (-8489, -8462, -9719, -28052))
      BEGIN
            -- Insert call to usp_ResendPending here
      END
           
      -- END CONVERSATION has to be issued AFTER the call to usp_ResendPending
      END CONVERSATION @handle;
END
</code></pre><p __designer:dtid="281474976711407">Resending on any of the other errors would most likely result immediately in an error being sent back because the condition that caused the first error would simply repeat itself, being SEND access denied or service contract agreement violation or whatever.</p> <h2 __designer:dtid="281474976711408" style="margin: 12pt 0in 3pt"><em __designer:dtid="281474976711409"><span __designer:dtid="281474976711410" style="font-size: 14pt; font-family: Arial">Idempotent Messages</span></em></h2> <p _\_designer:dtid="281474976711411">Ultimately a warning for caution if you choose to resend pending messages. The fact that a message is pending in sys.transmission\_queue is <strong __designer:dtid="281474976711412">not</strong> a guarantee that the target has not received that message. It only indicates that an acknowledgment was not received back and/or the sender did not yet delete the pending message. So if you decide to add resending logic in your application be careful that the sender is now prone to receiving duplicate data as the same message payload is successfully delivered <strong __designer:dtid="281474976711413">twice</strong>. One way to achieve this is to make the payload your messages carry idempotent, but how to achieve this is completely application specific, so I cannot give any generic advice on this (but hey, you could choose to use my consulting services and get a problem specific answer <span __designer:dtid="281474976711414" style="font-family: Wingdings"><span __designer:dtid="281474976711415">J</span></span>).</p>