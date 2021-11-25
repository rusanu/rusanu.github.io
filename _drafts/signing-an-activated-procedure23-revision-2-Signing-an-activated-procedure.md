---
id: 238
title: Signing an activated procedure
date: 2008-10-26T06:42:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/10/26/23-revision-2/
permalink: /2008/10/26/23-revision-2/
---
<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">In my previous entry </font><a href="http://blogs.msdn.com/remusrusanu/archive/2006/01/12/512085.aspx"><font face="Courier New" size="2">http://blogs.msdn.com/remusrusanu/archive/2006/01/12/512085.aspx</font></a><font face="Courier New" size="2"> I mentioned that one way to enable an activated procedure to be trusted at the server level is to sign the stored procedure with a certificate and grant appropriate permissions to the signing certificate. In this entry I want to show the details how to do this.</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p></o:p><!--more-->
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">Let’s say that we are implementing a SQL Server management and monitoring system that is based on Service Broker. One useful piece of functionality would be to be able to interrogate a SQL Server instance about existing sessions. A trivial implementation of this would be a service that responds to a request message with a list of current sessions:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <em><span style="font-family: Arial"><font size="2">Note: For simplicity, these scripts assume you have all necessary permissions. To get the expected result, you would have to run them as <strong>sysadmin</strong>.<o:p></o:p></font></span></em>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create a database for our demo<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create the service interface<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [RequestSessions] VALIDATION <span style="color: gray">=</span> EMPTY<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [Sessions] VALIDATION <span style="color: gray">=</span> WELL_FORMED_XML<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CONTRACT</span> [SessionsContract] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[RequestSessions] SENT <span style="color: blue">BY</span> INITIATOR<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Sessions] SENT <span style="color: blue">BY</span> TARGET<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create the service that exposes the interface<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [SessionsQueue]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">SERVICE</span> [SessionsService] <span style="color: blue">ON</span> <span style="color: blue">QUEUE</span> [SessionsQueue]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>[SessionsContract]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">Next step is to create a procedure that will implement the SessionsService:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create a procedure that implements<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; the [SessionsService] service<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS</span><span style="font-size: 10pt; font-family: 'Courier New'"> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SET</span> <span style="color: blue">NOCOUNT</span> <span style="color: blue">ON</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @mt <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>@dh <span style="color: gray">IS</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; If the message is a request message,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; send back a response with the list of sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span>@mt <span style="color: gray">=</span> N<span style="color: red">&#8216;RequestSessions&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Get the list of current sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; and send it back as a response<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @response <span style="color: blue">XML</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @response <span style="color: gray">=</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.dm_exec_sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> <span style="color: blue">XML</span> <span style="color: blue">PATH</span> <span style="color: gray">(</span><span style="color: red">&#8216;session&#8217;</span><span style="color: gray">),</span> <span style="color: blue">TYPE</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [Sessions]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>@response<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; End the conversation and commit<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Try to loop once more if there are message<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @dh <span style="color: gray">=</span> <span style="color: gray">NULL;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
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

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">Now we can go ahead and test our service. We need to create a service from which we can initiate a dialog with SessionsService and send a request for the current sessions:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [RequestsQueue]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">SERVICE</span> [Requests] <span style="color: blue">ON</span> <span style="color: blue">QUEUE</span> [RequestsQueue]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">Now initiatiate a dialog and send a request:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Requests]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> <span style="color: red">&#8216;SessionsService&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> [SessionsContract]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SEND</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [RequestSessions]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">To verify our implementation of the service, we can invoke the execute the procedure:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> SessionsServiceProcedure<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">This will create the response message and we can look for it in the [RequestsQueue]:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font size="2"><font face="Courier New"><span style="color: blue">RECEIVE</span><span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">)</span> <span style="color: blue">FROM</span> [RequestsQueue]</span></font></font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <span></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">We can see that all the response contains an XML response with the information for all the current sessions on the server. Since things are working fine, next step is to make the implementation procedure an activated procedure and attach it to the service queue:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [SessionsQueue] <span style="color: blue">WITH</span> ACTIVATION <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>STATUS <span style="color: gray">=</span> <span style="color: blue">ON</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>MAX_QUEUE_READERS <span style="color: gray">=</span> 1<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>PROCEDURE_NAME <span style="color: gray">=</span> [SessionsServiceProcedure]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <span><font size="2"><font face="Courier New"><span> </span><span style="color: blue">EXECUTE</span> <span style="color: blue">AS</span> OWNER<span style="color: gray">);<o:p></o:p></span></font></font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <span style="color: gray"><o:p><font face="Courier New" size="2"> </font></o:p></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">Now if we test by sending again a request, we’ll get the response immediately because of the activation:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Requests]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> <span style="color: red">&#8216;SessionsService&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> [SessionsContract]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SEND</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [RequestSessions]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font size="2"><font face="Courier New"><span style="color: blue">WAITFOR</span><span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">)</span> <span style="color: blue">FROM</span> [RequestsQueue]<span style="color: gray">);<o:p></o:p></span></span></font></font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">We can see that we get a response, but when we look at the response XML we’re in for the surprise: it contains only one session, instead of all the current sessions on the server! This is because the <strong><span style="color: blue">VIEW SERVER STATE</span></strong> privilege is required to view <strong>all</strong> the sessions other than the current one. When activated, the procedure has lost the necessary privileges and it can only see its own sessions. The reason why this happens is explained in my previous article at </font><a href="http://blogs.msdn.com/remusrusanu/archive/2006/01/12/512085.aspx"><font face="Courier New" size="2">http://blogs.msdn.com/remusrusanu/archive/2006/01/12/512085.aspx</font></a><font face="Courier New" size="2"> </font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">One possible solution is to simply mark the [<span>DemoActivationSigning</span>] database as trustworthy. But if we need finer grained control, we cannot do this. Marking the database as trustworthy makes the members of the database db_owner role de facto sysadmins. I’ll show here the (complex) way of achieving what we need using code signing. If you are a sysadmin asked to solve this problem, these are the steps needed to solve the problem:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Courier New" size="2">inspect the procedure code to ensure that you trust it</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font face="Courier New" size="2">change the procedure to have an EXECUTE AS OWNER clause</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><font size="2"><font face="Courier New">create a certificate with a private key in the [<span>DemoActivationSigning] database</span></font></font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">sign the procedure with the private key of the certificate you created</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">drop the private key of the certificate (to prevent it from ever being used again)</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">copy the certificate into the master database</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">create a login from the certificate</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">grant AUTHENTICATE SERVER to the certificate derived login</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span><span><font face="Courier New" size="2">&#8211;</font><span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span><font face="Courier New" size="2">grant any additional priviledge required by the procedure (e.g. <st1:place w:st="on"><st1:placename w:st="on">VIEW</st1:placename> <st1:placename w:st="on">SERVER</st1:placename> <st1:placetype w:st="on">STATE</st1:placetype></st1:place>) to the certificate derived login</font></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <span><o:p><font face="Courier New" size="2"> </font></o:p></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">Here is the code to achieve this:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Alter the procedure to add an EXECUTE AS OWNER clause.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; The procedure must have an EXECUTE AS clause for<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; the code signing infrastructure to work properly.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> <span style="color: blue">EXECUTE</span> <span style="color: blue">AS</span> OWNER<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS</span><span style="font-size: 10pt; font-family: 'Courier New'"> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SET</span> <span style="color: blue">NOCOUNT</span> <span style="color: blue">ON</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @mt <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>@dh <span style="color: gray">IS</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; If the message is a request message,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; send back a response with the list of sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span>@mt <span style="color: gray">=</span> N<span style="color: red">&#8216;RequestSessions&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Get the list of current sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; and send it back as a response<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @response <span style="color: blue">XML</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @response <span style="color: gray">=</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.dm_exec_sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> <span style="color: blue">XML</span> <span style="color: blue">PATH</span> <span style="color: gray">(</span><span style="color: red">&#8216;session&#8217;</span><span style="color: gray">),</span> <span style="color: blue">TYPE</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [Sessions]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>@response<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; End the conversation and commit<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Try to loop once more if there are message<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @dh <span style="color: gray">=</span> <span style="color: gray">NULL;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create a certificate with a private key <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; to sign the procedure with. The password<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; used is not important, we&#8217;ll drop the <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; private key<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>ENCRYPTION <span style="color: blue">BY</span> PASSWORD <span style="color: gray">=</span> <span style="color: red">&#8216;Password#1234&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> SUBJECT <span style="color: gray">=</span> <span style="color: red">&#8216;SessionsServiceProcedure Signing certificate&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Sign the procedure with the certificate&#8217;s private key<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ADD</span><span style="font-size: 10pt; font-family: 'Courier New'"> SIGNATURE <span style="color: blue">TO</span> OBJECT<span style="color: gray">::</span>[SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BY</span> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> PASSWORD <span style="color: gray">=</span> <span style="color: red">&#8216;Password#1234&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Drop the private key. This way it cannot be<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; used again to sign other procedures.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>REMOVE PRIVATE <span style="color: blue">KEY</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Copy the certificate in [master]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; We must backup to a file and create<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; the certificate in [master] from this file<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BACKUP</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">FILE</span> <span style="color: gray">=</span> <span style="color: red">&#8216;C:SessionsServiceProcedure.CER&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> MASTER<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">FILE</span> <span style="color: gray">=</span> <span style="color: red">&#8216;C:SessionsServiceProcedure.CER&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create a login from the certificate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">LOGIN</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedure]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Grant the reuired priviledges to the certificate login<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; The AUTHENTICATE SERVER permission is required for <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; any other permission to take effect<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">GRANT</span><span style="font-size: 10pt; font-family: 'Courier New'"> AUTHENTICATE <span style="color: blue">SERVER</span> <span style="color: blue">TO</span> [SessionsServiceProcedure]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <st1:place w:st="on"><st1:placename w:st="on"><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">GRANT</span></st1:placename><span style="font-size: 10pt; font-family: 'Courier New'"> <st1:placename w:st="on"><span style="color: blue">VIEW</span></st1:placename> <st1:placename w:st="on"><span style="color: blue">SERVER</span></st1:placename> <st1:placetype w:st="on">STATE</st1:placetype></span></st1:place><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TO</span> [SessionsServiceProcedure]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">We can now go back and try again to send a message:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Requests]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> <span style="color: red">&#8216;SessionsService&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> [SessionsContract]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SEND</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [RequestSessions]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">WAITFOR</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">)</span> <span style="color: blue">FROM</span> [RequestsQueue]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This time the result is the one expected, the response message contains all the sessions.
</p>