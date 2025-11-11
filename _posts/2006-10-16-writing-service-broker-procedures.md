---
id: 28
title: Writing Service Broker Procedures
date: 2006-10-16T03:44:18+00:00
author: remus
layout: post
guid: /2006/10/16/writing-service-broker-procedures/
permalink: /2006/10/16/writing-service-broker-procedures/
categories:
  - Samples
  - Tutorials
---
It’s been a while since I posted an entry in this blog and this article is long overdue. There were a series of events that prevented me from posting this, not least impacting being the fact that I’ve opened a WoW account…

<h1 style="margin: 12pt 0in 3pt">
  <font face="Arial" size="5">T-SQL RECEIVE. Fast.</font>
</h1>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  A question often asked is how to write a typical activated procedure? This article will cover the ways to write a performant T-SQL procedure to process messages. I am not going to cover CLR procedures for now.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <!--more-->
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Basic Receive</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The typical example shows how to do this by creating a WHILE loop and the RECEIVE TOP (1) the next message from the queue and process it, each message in an individual transaction:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [BasicReceive]<o:p></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @messageTypeName <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span><span style="color: gray">(</span>1<span style="color: gray">)</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@h <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageTypeName <span style="color: gray">=</span> message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@payload <span style="color: gray">=</span> message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Some basic processing. Send back an echo reply<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @h <span style="color: gray">(</span>@payload<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Log the received error into ERRORLOG and system Event Log (eventvwr.exe)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h_string <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error_message <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @h_string <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@h <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">)),</span> @error_message <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@payload <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Conversation %s was ended with error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @h_string<span style="color: gray">,</span> @error_message<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The question on the table is: is this the fastest way to RECEIVE messages from a queue? Well, let’s measure it how fast it is.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Methodology</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  First thing we need is to devise a method to measure the performance of this procedure. My proposal is to use a simple way: preload the queue with a number of messages, and then run the procedure and measure how long it takes to drain the queue. I’m going to do these measurements on my home system, which is a measly single proc P4 2.4GHz with 1 GB of RAM. The storage consists of an 80GB WD and a 160 GB Maxtor IDE drives. I’ll store the MDF on the first and the LDF on the second. Of course, as with any such measurement, I’ll pre-grow the database and log files to an acceptable size to prevent dramatic alteration of results due to file growth events. Next thing we’ll do is to create a queue and load it with 100 conversations each with 100 messages (for a total of 10000 messages). For this, I’ll create a procedure that loads the queue:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.databases</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> <span style="color: red">&#8216;ReceivePerfBlog&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">DATABASE</span> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [ReceivePerfBlog]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ON</span> <span style="color: gray">(</span><span style="color: blue">NAME</span> <span style="color: gray">=</span> ReceivePerfBlog<span style="color: gray">,</span> <span style="color: blue">FILENAME</span> <span style="color: gray">=</span> <span style="color: red">&#8216;C:\Program Files\Microsoft SQL Server\MSSQL.1\MSSQL\Data\ReceivePerfBlog.MDF&#8217;</span><span style="color: gray">,</span> <span style="color: blue">SIZE</span> <span style="color: gray">=</span> 4GB<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><st1:place w:st="on"><st1:city w:st="on"><span style="color: fuchsia">LOG</span></st1:city> <st1:state w:st="on"><span style="color: blue">ON</span></st1:state></st1:place> <span style="color: gray">(</span><span style="color: blue">NAME</span> <span style="color: gray">=</span> ReceivePerfBlog_log<span style="color: gray">,</span> <span style="color: blue">FILENAME</span> <span style="color: gray">=</span> <span style="color: red">&#8216;E:\DATA\ReceivePerfBlog.LDF&#8217;</span><span style="color: gray">,</span> <span style="color: blue">SIZE</span> <span style="color: gray">=</span> 10GB<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [Initiator]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [Target]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">SERVICE</span> [Initiator] <span style="color: blue">ON</span> <span style="color: blue">QUEUE</span> [Initiator]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">SERVICE</span> [Target] <span style="color: blue">ON</span> <span style="color: blue">QUEUE</span> [Target] <span style="color: gray">(</span>[DEFAULT]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; This procedure loads the test queue qith the <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<span> </span>number of messages and conversations passed in<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> LoadQueueReceivePerfBlog<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@conversationCount <span style="color: blue">INT</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messagesPerConversation <span style="color: blue">INT</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">)<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @batchCount <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @batchCount <span style="color: gray">=</span> 0<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> @conversationCount <span style="color: gray">></span> 0<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @h<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Initiator]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> N<span style="color: red">&#8216;Target&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;current database&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @messageCount <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @messageCount <span style="color: gray">=</span> 0<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> @messageCount <span style="color: gray"><</span> @messagesPerConversation<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @h <span style="color: gray">(</span>@payload<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @messageCount <span style="color: gray">=</span> @messageCount <span style="color: gray">+</span> 1<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@batchCount <span style="color: gray">=</span> @batchCount <span style="color: gray">+</span> 1<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @batchCount <span style="color: gray">>=</span> 100<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @batchCount <span style="color: gray">=</span> 0<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @conversationCount <span style="color: gray">=</span> @conversationCount<span style="color: gray">&#8211;</span>1<o:p></o:p></span>
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

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Measuring the performance</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  First procedure we’re going to measure is the very basic RECEIVE TOP (1) that processes on single message per transaction.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>N<span style="color: red">&#8216;<Test/>&#8217;</span> <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> LoadQueueReceivePerfBlog 100<span style="color: gray">,</span>100<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: blue">FLOAT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: gray">=</span> <span style="color: fuchsia">COUNT</span><span style="color: gray">(*)</span> <span style="color: blue">FROM</span> [Target]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> [BasicReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">as</span> [Start]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@endTime <span style="color: blue">as</span> [End]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount <span style="color: blue">as</span> [Count]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>second<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)</span> <span style="color: blue">as</span> [Duration]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount<span style="color: gray">/</span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>millisecond<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)*</span>1000 <span style="color: blue">as</span> [Rate]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  These are the reported results on my machine:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 13:22:25.060 2006-10-14 13:22:57.107 10000<span> </span>32<span> </span>312.051426075017<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So the basic procedure can process about 312 msgs/sec. This in itself is no a bad result, but let’s see if we can go higher.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Batched Commits</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The first thing we can look at is to change the one message per transaction into batching multiple messages into one transaction. This is the most simple and basic optimization any database application should think of first, as the commit rate is probably the very first bottle necks any system will hit, especially on commodity hardware. So we’ll do a simple modification to our basic RECEIVE procedure: we’ll keep a counter of messages processed and commit only after, say, 100 messages were processed. Everything else in the processing stays the same:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.procedures</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;BatchedReceive&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [BatchedReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [BatchedReceive]<o:p></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @messageTypeName <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">DECLARE</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> @batchCount <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @batchCount <span style="color: gray">=</span> 0<span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: gray; font-family: 'Courier New'"><o:p> </o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;</span></span></strong><strong><span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span><span style="color: gray">(</span>1<span style="color: gray">)</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@h <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@messageTypeName <span style="color: gray">=</span> message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@payload <span style="color: gray">=</span> message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span> </span><span style="color: blue">BREAK</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Some basic processing. Send back an echo reply<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @h <span style="color: gray">(</span>@payload<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Log the received error into ERRORLOG and system Event Log (eventvwr.exe)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h_string <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error_message <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @h_string <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@h <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">)),</span> @error_message <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@payload <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Conversation %s was ended with error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @h_string<span style="color: gray">,</span> @error_message<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Increment the batch count and commit every 100 messages<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">SELECT</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> @batchCount <span style="color: gray">=</span> @batchCount <span style="color: gray">+</span> 1<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> @batchCount <span style="color: gray">>=</span> 100<o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @batchCount <span style="color: gray">=</span> 0<span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;</span></span></strong><strong><span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  I have highlighted the parts that changed in the procedure code. Let’s go ahead, load again 10000 messages in the queue and measure how fast they are drained:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [ReceivePerfBlog] <span style="color: blue">SET</span> NEW_BROKER <span style="color: blue">WITH</span> <span style="color: blue">ROLLBACK</span> IMMEDIATE<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>N<span style="color: red">&#8216;<Test/>&#8217;</span> <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> LoadQueueReceivePerfBlog 100<span style="color: gray">,</span>100<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: blue">FLOAT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: gray">=</span> <span style="color: fuchsia">COUNT</span><span style="color: gray">(*)</span> <span style="color: blue">FROM</span> [Target]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> [BatchedReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">as</span> [Start]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@endTime <span style="color: blue">as</span> [End]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount <span style="color: blue">as</span> [Count]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>second<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)</span> <span style="color: blue">as</span> [Duration]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount<span style="color: gray">/</span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>millisecond<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)*</span>1000 <span style="color: blue">as</span> [Rate]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The results are:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 13:37:36.497 2006-10-14 13:38:02.560 10000<span> </span>26<span> </span>383.685684687104<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So we increased the processing rate at 383 msgs/sec, about 22% faster than the single message per transaction procedure. The result is better, but not earth shattering better. Mostly this is because I already took the precaution of separating the LDF file into a disk that has no other activity other that this LDF file, so streaming in the log pages is quite fast.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Cursor based processing</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The next step will take is a departure from the basic RECEIVE procedure. Instead of using the TOP (1) clause, we’ll use a cursor and process as many messages as RECEIE returns in one execution. Because the T-SQL cursors unfortunately cannot be declared on top of the RECEIVE resultset, we’re going to use a trick: RECEIVE into a table variable, then iterate the table variable using a cursor. The processing for each message will be identical as for the previous cases.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.procedures</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;CursorReceive&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [CursorReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [CursorReceive]<o:p></o:p></span>
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
  <strong><span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">DECLARE</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> @tableMessages <span style="color: blue">TABLE</span> <span style="color: gray">(<o:p></o:p></span></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>queuing_order <span style="color: blue">BIGINT</span><span style="color: gray">,<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>conversation_handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">,<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>message_type_name <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>message_body <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Create cursor over the table variable<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Use the queueing_order column to <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; preserve the message order<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> cursorMessages <o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">CURSOR</span> FORWARD_ONLY READ_ONLY<o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> <span style="color: blue">SELECT</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>message_type_name<span style="color: gray">,<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span>message_body<o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> @tableMessages<o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">ORDER</span> <span style="color: blue">BY</span> queuing_order<span style="color: gray">;</span></span></strong><strong><span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @messageTypeName <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @tableMessages<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
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
  <strong><span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">OPEN</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> cursorMessages<span style="color: gray">;<o:p></o:p></span></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">FETCH</span> NEXT <span style="color: blue">FROM</span> cursorMessages<o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @h<span style="color: gray">,</span> @messageTypeName<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@FETCH_STATUS</span> <span style="color: gray">!=</span> 0<span style="color: gray">)<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">BREAK</span><span style="color: gray">;</span></span></strong><strong><span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Some basic processing. Send back an echo reply<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> N<span style="color: red">&#8216;DEFAULT&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @h <span style="color: gray">(</span>@payload<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> N<span style="color: red">&#8216;http://schemas.microsoft.com/SQL/ServiceBroker/Error&#8217;</span> <span style="color: gray">=</span> @messageTypeName<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Log the received error into ERRORLOG and system Event Log (eventvwr.exe)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h_string <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error_message <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @h_string <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@h <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">)),</span> @error_message <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@payload <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Conversation %s was ended with error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @h_string<span style="color: gray">,</span> @error_message<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; color: blue">CLOSE</span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial"> cursorMessages<span style="color: gray">;<o:p></o:p></span></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> @tableMessages<span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span style="background: yellow none repeat scroll 0% 50%; font-size: 10pt; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial; font-family: 'Courier New'"><span> </span><span style="color: blue">DEALLOCATE</span> cursorMessages<span style="color: gray">;</span></span></strong><strong><span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p></o:p></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Again, I highlighted the significant changes on the procedure.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Let’s run this and see the results:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [ReceivePerfBlog] <span style="color: blue">SET</span> NEW_BROKER <span style="color: blue">WITH</span> <span style="color: blue">ROLLBACK</span> IMMEDIATE<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>N<span style="color: red">&#8216;<Test/>&#8217;</span> <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> LoadQueueReceivePerfBlog 100<span style="color: gray">,</span>100<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: blue">FLOAT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @msgCount <span style="color: gray">=</span> <span style="color: fuchsia">COUNT</span><span style="color: gray">(*)</span> <span style="color: blue">FROM</span> [Target]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> [CursorReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @endTime <span style="color: gray">=</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">();<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @startTime <span style="color: blue">as</span> [Start]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@endTime <span style="color: blue">as</span> [End]<span style="color: gray">,</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount <span style="color: blue">as</span> [Count]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>second<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)</span> <span style="color: blue">as</span> [Duration]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@msgCount<span style="color: gray">/</span><span style="color: fuchsia">DATEDIFF</span><span style="color: gray">(</span>millisecond<span style="color: gray">,</span> @startTime<span style="color: gray">,</span> @endTime<span style="color: gray">)*</span>1000 <span style="color: blue">as</span> [Rate]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 14:08:01.623 2006-10-14 14:08:07.590 10000<span> </span>6<span> </span>1676.16493462957<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Now we’re talking! This kind of processing improved our performance to 1676 msgs/sec, a 437% improvement. BTW, if you’re asking yourself what happened to the batch commit, is still there. Because the procedure processes one entire RECEIVE resultset in a transaction, the batch commit is inherent.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  With this procedure we pretty much pushed the performance of a message by message processing as far as it goes. The procedure can be further tweaked, but will not give significant performance boosts. For instance, we can replace the message type name with the message type id. This will be a faster comparison (int vs. string) and also will prevent RECEIVE from joining into it’s plan the sys.service_message_types view.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.procedures</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;CursorMessageTypeReceive&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [CursorMessageTypeReceive]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [CursorMessageTypeReceive]<o:p></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @tableMessages <span style="color: blue">TABLE</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order <span style="color: blue">BIGINT</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">message_type <span style="color: blue">INT</span><span style="color: gray">,</span></span><span style="color: gray"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Create cursor over the table variable<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Use the queueing_order column to <o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; preserve the message order<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> cursorMessages <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">CURSOR</span> FORWARD_ONLY READ_ONLY<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> <span style="color: blue">SELECT</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> @tableMessages<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ORDER</span> <span style="color: blue">BY</span> queuing_order<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> <span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">@messageType <span style="color: blue">INT</span><span style="color: gray">;</span></span><span style="color: gray"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">message_type_id<span style="color: gray">,</span></span><span style="color: gray"><o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @tableMessages<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">OPEN</span> cursorMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FETCH</span> NEXT <span style="color: blue">FROM</span> cursorMessages<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @h<span style="color: gray">,</span> @messageType<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@FETCH_STATUS</span> <span style="color: gray">!=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BREAK</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Some basic processing. Send back an echo reply<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">14 <span style="color: gray">=</span> @messageType</span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @h <span style="color: gray">(</span>@payload<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> <span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">2 <span style="color: gray">=</span> @messageType</span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ELSE</span> <span style="color: blue">IF</span> <span style="background: yellow none repeat scroll 0% 50%; -moz-background-clip: -moz-initial; -moz-background-origin: -moz-initial; -moz-background-inline-policy: -moz-initial">1 <span style="color: gray">=</span> @messageType</span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Log the received error into ERRORLOG and system Event Log (eventvwr.exe)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @h_string <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @error_message <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @h_string <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@h <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>100<span style="color: gray">)),</span> @error_message <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@payload <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>4000<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RAISERROR</span> <span style="color: gray">(</span>N<span style="color: red">&#8216;Conversation %s was ended with error %s&#8217;</span><span style="color: gray">,</span> 10<span style="color: gray">,</span> 1<span style="color: gray">,</span> @h_string<span style="color: gray">,</span> @error_message<span style="color: gray">)</span> <span style="color: blue">WITH</span> <span style="color: fuchsia">LOG</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @h<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">CLOSE</span> cursorMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> @tableMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DEALLOCATE</span> cursorMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO</span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  With these improvements, the results are:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 14:04:53.967 2006-10-14 14:04:59.763 10000<span> </span>6<span> </span>1725.32781228433<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  This gives only a 3% improvement, and makes the procedure more difficult to maintain and debug because it uses the message types IDs instead of the names. I’m not convinced the benefits justify the potential problems.
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Set based Processing</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The next step we can do is not a free one. We’re going to move away from the message by message processing and see how can we do set based processing. One cautionary note is required here: not all messaging applications are going to be able to do a set based processing for the incoming messages. But certain messaging patterns are particularly well fit for this kind of processing. Whenever one conversation side has to send long streams of messages w/o a response from the other side, this kind of processing usually can be applied. A good example of such pattern is <strong>audit</strong>: front end machines have to record user actions for audit needs. Rather than connecting to a central database and inserting directly the audit record into the database, they start a one directional conversation on which they’ll send the audit data as messages. The back-end processing of the message is straightforward: extract the audit data from the message payload and insert it into the audit tables. This can be done as a set based operation, the entire RECEIVE resultset can be inserted as a single set into the audit table.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">XML payload</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  So let’s create a dummy audit like back-end: the messages consist of an XML payload containing the user name, the date and some arbitrary audit payload. The back end has to store these records into a table, shredding the XML into relational columns first. As with any well planned application that does this, it should also store the original XML received.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [ReceivePerfBlog] <span style="color: blue">SET</span> NEW_BROKER <span style="color: blue">WITH</span> <span style="color: blue">ROLLBACK</span> IMMEDIATE<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [ReceivePerfBlog]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.tables</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;PayloadData&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">TABLE</span> [PayloadData]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TABLE</span> [PayloadData] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Id] <span style="color: blue">INT</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL</span> <span style="color: blue">IDENTITY</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[DateTime] <span style="color: blue">DATETIME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Payload] <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[User] <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>256<span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Original] <span style="color: blue">XML</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.procedures</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;RowsetDatagram&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [RowsetDatagram]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [RowsetDatagram]<o:p></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @tableMessages <span style="color: blue">TABLE</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order <span style="color: blue">BIGINT</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type_name <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>payload <span style="color: blue">XML</span><span style="color: gray">);</span><span style="color: green">&#8211;([DatagramSchemaCollection]));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">)</span> <span style="color: blue">AS</span> payload<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @tableMessages<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Rowset based datagram processing:<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Extract the XML attributes and insert into table<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">;</span><span style="color: blue">WITH</span> <span style="color: blue">XMLNAMESPACES</span> <span style="color: gray">(</span><span style="color: blue">DEFAULT</span> <span style="color: red">&#8216;http://tempuri.org/RemusRusanu/Blog/10/14/2006/Datagram&#8217;</span><span style="color: gray">)</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INSERT</span> <span style="color: blue">INTO</span> [PayloadData] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>[DateTime]<span style="color: gray">,</span> [Payload]<span style="color: gray">,</span> [User]<span style="color: gray">,</span> [Original]<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> payload<span style="color: gray">.</span><span style="color: blue">value</span><span style="color: gray">(</span>N<span style="color: red">&#8216;(/Datagram/@date-time)[1]&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;DATETIME&#8217;</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>payload<span style="color: gray">.</span><span style="color: blue">value</span><span style="color: gray">(</span>N<span style="color: red">&#8216;(/Datagram/@payload)[1]&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;NVARCHAR(MAX)&#8217;</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>payload<span style="color: gray">.</span><span style="color: blue">value</span><span style="color: gray">(</span>N<span style="color: red">&#8216;(/Datagram/@user)[1]&#8217;</span><span style="color: gray">,</span> <span style="color: red">&#8216;NVARCHAR(256)&#8217;</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>payload<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> @tableMessages<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> message_type_name <span style="color: gray">=</span> N<span style="color: red">&#8216;DEFAULT&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">ORDER</span> <span style="color: blue">BY</span> queuing_order<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> @tableMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'">To test this procedure, we’re gonna have to preload the queue with valid XML payloads:<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @xmlPayload <span style="color: blue">XML</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">WITH</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">XMLNAMESPACES</span> <span style="color: gray">(</span><span style="color: blue">DEFAULT</span> <span style="color: red">&#8216;http://tempuri.org/RemusRusanu/Blog/10/14/2006/Datagram&#8217;</span><span style="color: gray">)</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @xmlPayload <span style="color: gray">=</span> <span style="color: gray">(</span><span style="color: blue">SELECT</span> <span style="color: fuchsia">GETDATE</span><span style="color: gray">()</span> <span style="color: blue">AS</span> [@date-time]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">SUSER_SNAME</span><span style="color: gray">()</span> <span style="color: blue">AS</span> [@user]<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>N<span style="color: red">&#8216;Some Data&#8217;</span> <span style="color: blue">AS</span> [@payload]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FOR</span> <span style="color: blue">XML</span> <span style="color: blue">PATH</span><span style="color: gray">(</span>N<span style="color: red">&#8216;Datagram&#8217;</span><span style="color: gray">),</span> <span style="color: blue">TYPE</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SELECT</span><span style="font-size: 10pt; font-family: 'Courier New'"> @payload <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@xmlPayload <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">EXEC</span><span style="font-size: 10pt; font-family: 'Courier New'"> LoadQueueReceivePerfBlog 100<span style="color: gray">,</span>100<span style="color: gray">,</span> @payload<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Here are the results for this procedure:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 14:55:23.827 2006-10-14 14:55:28.733 10000<span> </span>5<span> </span>2038.32042397065<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  We managed to boost the numbers up to above 2000 msgs/sec! Not bad for commodity hardware, like my home system.
</p>

<h3 style="margin: 12pt 0in 3pt">
  <font face="Arial">Binary Payload</font>
</h3>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  We all love and like XML, but at the end of the day XML is text all too fond of brackets. How much could we improve the performance by moving to a binary payload for these audit records? Now doing binary marshaling and un-marshaling in T-SQL code is not for the faint of heart, but I’ve seen braver things. The first thing we need is to create two functions, one that marshals the audit data into a binary blob and one that un-marshals out the original data from a blob:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">FUNCTION</span> [BinaryMarhsalPayload] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@dateTime <span style="color: blue">DATETIME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@user <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>256<span style="color: gray">))<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RETURNS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @marshaledPayload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payloadLength <span style="color: blue">BIGINT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @userLength <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @payloadLength <span style="color: gray">=</span> <span style="color: fuchsia">LEN</span><span style="color: gray">(</span>@payload<span style="color: gray">),</span> @userLength <span style="color: gray">=</span> <span style="color: fuchsia">LEN</span><span style="color: gray">(</span>@user<span style="color: gray">)*</span>2<span style="color: gray">;</span> <span style="color: green">&#8212; wchar_t<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @marshaledPayload <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@dateTime <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">))</span> <span style="color: gray">+<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@payloadLength <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">))</span> <span style="color: gray">+</span> @payload <span style="color: gray">+<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@userLength <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">))</span> <span style="color: gray">+</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>@user <span style="color: blue">AS</span> <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RETURN</span> <span style="color: gray">(</span>@marshaledPayload<span style="color: gray">);<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS</span> <span style="color: gray">(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.objects</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;BinaryUnmarhsalPayload&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">FUNCTION</span> [BinaryUnmarhsalPayload]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">FUNCTION</span> [BinaryUnmarhsalPayload] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>@message_body <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">))<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RETURNS</span> @unmarshaledBody <span style="color: blue">TABLE</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[DateTime] <span style="color: blue">DATETIME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Payload] <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[User] <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>256<span style="color: gray">))<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @dateTime <span style="color: blue">DATETIME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @user <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payload <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @payloadLength <span style="color: blue">BIGINT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @userLength <span style="color: blue">INT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @dateTime <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span><span style="color: fuchsia">SUBSTRING</span><span style="color: gray">(</span>@message_body<span style="color: gray">,</span> 1<span style="color: gray">,</span> 8<span style="color: gray">)</span> <span style="color: blue">AS</span> <span style="color: blue">DATETIME</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @payloadLength <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span><span style="color: fuchsia">SUBSTRING</span><span style="color: gray">(</span>@message_body<span style="color: gray">,</span> 9<span style="color: gray">,</span> 8<span style="color: gray">)</span> <span style="color: blue">AS</span> <span style="color: blue">BIGINT</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @payload <span style="color: gray">=</span> <span style="color: fuchsia">SUBSTRING</span><span style="color: gray">(</span>@message_body<span style="color: gray">,</span> 17<span style="color: gray">,</span> @payloadLength<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @userLength <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span><span style="color: fuchsia">SUBSTRING</span><span style="color: gray">(</span>@message_body<span style="color: gray">,</span> @payloadLength <span style="color: gray">+</span> 17<span style="color: gray">,</span> 4<span style="color: gray">)</span> <span style="color: blue">AS</span> <span style="color: blue">INT</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> @user <span style="color: gray">=</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span><span style="color: fuchsia">SUBSTRING</span><span style="color: gray">(</span>@message_body<span style="color: gray">,</span> @payloadLength <span style="color: gray">+</span> 21<span style="color: gray">,</span> @userLength<span style="color: gray">)</span> <span style="color: blue">AS</span> <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>256<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INSERT</span> <span style="color: blue">INTO</span> @unmarshaledBody <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">VALUES</span> <span style="color: gray">(</span>@dateTime<span style="color: gray">,</span> @payload<span style="color: gray">,</span> @user<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">RETURN</span><span style="color: gray">;<o:p></o:p></span></span>
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
  We can now create our test procedure:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.tables</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;PayloadData&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">TABLE</span> [PayloadData]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">TABLE</span> [PayloadData] <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Id] <span style="color: blue">INT</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL</span> <span style="color: blue">IDENTITY</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[DateTime] <span style="color: blue">DATETIME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[Payload] <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">),<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>[User] <span style="color: blue">NVARCHAR</span><span style="color: gray">(</span>256<span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">IF</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">EXISTS(</span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.procedures</span> <span style="color: blue">WHERE</span> <span style="color: blue">NAME</span> <span style="color: gray">=</span> N<span style="color: red">&#8216;RowsetBinaryDatagram&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DROP</span> <span style="color: blue">PROCEDURE</span> [RowsetBinaryDatagram]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [RowsetBinaryDatagram]<o:p></o:p></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DECLARE</span> @tableMessages <span style="color: blue">TABLE</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order <span style="color: blue">BIGINT</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type_name <span style="color: blue">SYSNAME</span><span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body <span style="color: blue">VARBINARY</span><span style="color: gray">(</span><span style="color: fuchsia">MAX</span><span style="color: gray">));<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>1<span style="color: gray">=</span>1<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WAITFOR</span><span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>queuing_order<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_type_name<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span>message_body<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> [Target]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INTO</span> @tableMessages<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">IF</span> <span style="color: gray">(</span><span style="color: fuchsia">@@ROWCOUNT</span> <span style="color: gray">=</span> 0<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
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
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Rowset based datagram processing:<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: green">&#8212; Unmarshal the binary result into the table<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">INSERT</span> <span style="color: blue">INTO</span> [PayloadData] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">(</span>[DateTime]<span style="color: gray">,</span> [Payload]<span style="color: gray">,</span> [User]<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">SELECT</span> [DateTime]<span style="color: gray">,</span> [Payload]<span style="color: gray">,</span> [User] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">FROM</span> @tableMessages<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: gray">CROSS</span> <span style="color: gray">APPLY</span> dbo<span style="color: gray">.</span>[BinaryUnmarhsalPayload]<span style="color: gray">(</span>message_body<span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">WHERE</span> message_type_name <span style="color: gray">=</span> <span style="color: red">&#8216;DEFAULT&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">DELETE</span> <span style="color: blue">FROM</span> @tableMessages<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span> </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Note the use of the <span style="font-family: 'Courier New'">CROSS APPLY</span> syntax to produce the function output as columns into the <span style="font-family: 'Courier New'">SELECT</span> projection. Here are the result numbers:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">Start<span> </span>End<span> </span>Count<span> </span>Duration<span> </span>Rate<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;- &#8212;&#8212;&#8212;&#8211; &#8212;&#8212;&#8212;&#8212;&#8212;&#8212;&#8212;-<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">2006-10-14 15:09:16.013 2006-10-14 15:09:19.700 10000<span> </span>3<span> </span>2712.96798697775<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'">(1 row(s) affected)<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Right, that is 2712 msgs/sec. This is how far my knowledge can get you. I don’t know of any faster way to process messages than this. But I’m sure I’ll soon be proven wrong by some intrepid user <span style="font-family: Wingdings"><span>J</span></span> Are you willing to pick up this challenge?
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Activation</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  I have measured the performance of the procedure by directly invoking the procedure. How is this different if activation would be involved? Not much. From performance point of view invoking a procedure from the activated context is not different that invoking it from a user session. But there are other things to consider:
</p>

<ul style="margin-top: 0in" type="disc">
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    Multiple CPU machines. Back-end machines are often multiple proc machines, usually in the 4-way and 8-way range. The more CPUs, the better, if you can pay the bill <span style="font-family: Wingdings"><span>J</span></span>. When hosting activation on such machines, usually you’d configure the queue<span> </span><span> </span>with a max_queue_readers to match the number of CPUs. The performance results we’ve seen today would scale nearly linearly with each new instance of the procedure added, provided one thing: there are enough distinct conversation groups in the queue to feed all the procedures. What happens is that each procedure would lock a conversation group to produce the RECEIVE resultset. Even if there are a million messages in the queue, if they are all on once conversation a second instance of the activated procedure would simply not have access to them!
  </li>
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    Activation warm up time. If you’re measuring activated procedures performance, you’d have to consider that new procedure instances are launched at most one every 5 seconds. So if you have an 8-way system with max_queue_readers = 8, it would take 35 seconds just to launch all the 8 procedures! Take this into consideration when measuring performance involving activation.
  </li>
</ul>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<h2 style="margin: 12pt 0in 3pt">
  <em><font face="Arial">Conclusions</font></em>
</h2>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  We showed how we can achieve results of processing more that 2500 msgs/sec on a really low end machine. The actual results you may get in your application, of course, will vary.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  While the situation tested is idealized (drain of a preloaded queue), often times in real production systems similar situations happen (e.g. after a spike of incoming messages, or after the processing service was stopped for maintenance for a period). Also, the best results we’ve seen are achievable only on specific message exchange patterns.
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The cursor based processing can perform only if the cost of setting up the cursor is largely outweigh by the cost of processing only one message (TOP 1) from a potential resultset of tens or hundreds of messages. If the message exchange pattern is request-reply, than a RECEIVE cannot return more than one message in a resultset (<em>the</em> request). In a request-reply case, the basic batched RECEIVE procedure might be the best performing one.
</p>