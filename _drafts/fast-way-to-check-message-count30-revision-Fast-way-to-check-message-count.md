---
id: 160
title: Fast way to check message count
date: 2007-10-29T12:51:56+00:00
author: remus
layout: revision
guid: http://rusanu.com/2007/10/29/30-revision/
permalink: /2007/10/29/30-revision/
---
<span style="font-size: 10pt; font-family: 'Courier New'"><span style="color: red"><font color="#000000"><span style="font-size: 10pt; font-family: Arial">A fast way to check the count of messages in a queue is to run this query is to look at the row count of the underlying </span><font face="Times New Roman" size="3">the b-tree that stores the messages</font><span style="font-size: 10pt; font-family: Arial">:<o:p></o:p></span></font></span></span>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">select</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> p</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">rows<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><span> </span>from</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: green; font-family: 'Courier New'">sys.objects</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">as</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> o <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><span> </span></span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">join</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: green; font-family: 'Courier New'">sys.partitions</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">as</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> p </span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">on</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> p</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: fuchsia; font-family: 'Courier New'">object_id</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">=</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> o</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: fuchsia; font-family: 'Courier New'">object_id</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><span> </span></span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">join</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: green; font-family: 'Courier New'">sys.objects</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">as</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> q </span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">on</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> o</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">parent_object_id </span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">=</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> q</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: fuchsia; font-family: 'Courier New'">object_id</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><span> </span>where</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> q</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: blue; font-family: 'Courier New'">name</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">=</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> </span><span style="font-size: 10pt; color: red; font-family: 'Courier New'">&#8216;<queuename>&#8217;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><span> </span></span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">and</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> p</span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">.</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'">index_id </span><span style="font-size: 10pt; color: gray; font-family: 'Courier New'">=</span><span style="font-size: 10pt; color: black; font-family: 'Courier New'"> 1</span><!--more-->
  
  <span style="color: blue"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font color="#000000" face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font color="#000000" face="Times New Roman" size="3">This query is <strong>significantly</strong> faster than running a COUNT(*), specially on a busy system where many rows may be locked. In fact this can be used to retrieve also other counts fast:</font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font size="3"><strong><font color="#000000" face="Times New Roman">Messages pending in transmission_queue:<br /> </font></strong><span style="color: blue; font-family: 'Courier New'">select</span><span style="font-family: 'Courier New'"><font color="#000000"> p</font><span style="color: gray">.</span><font color="#000000">rows <o:p></o:p></font></span></font>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: blue">from</span><font color="#000000"> </font><span style="color: green">sys.objects</span><font color="#000000"> </font><span style="color: blue">as</span><font color="#000000"> o </font></span><span style="font-size: 10pt; font-family: 'Courier New'"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: gray">join</span><font color="#000000"> </font><span style="color: green">sys.partitions</span><font color="#000000"> </font><span style="color: blue">as</span><font color="#000000"> p </font><span style="color: blue">on</span><font color="#000000"> p</font><span style="color: gray">.</span><span style="color: fuchsia">object_id</span><font color="#000000"> </font><span style="color: gray">=</span><font color="#000000"> o</font><span style="color: gray">.</span><span style="color: fuchsia">object_id</span><font color="#000000"> <o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: blue">where</span><font color="#000000"> o</font><span style="color: gray">.</span><span style="color: blue">name</span><font color="#000000"> </font><span style="color: gray">=</span><font color="#000000"> </font><span style="color: red">&#8216;sysxmitqueue&#8217;</span></span><span style="font-family: 'Courier New'"><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span><o:p><font color="#000000" face="Times New Roman" size="3"> </font></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <strong><span><font size="3"><font color="#000000"><font face="Times New Roman">Total number of conversations in the database:<o:p></o:p></font></font></font></span></strong>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">select</span><span style="font-size: 10pt; font-family: 'Courier New'"><font color="#000000"> p</font><span style="color: gray">.</span><font color="#000000">rows<o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: blue">from</span><font color="#000000"> </font><span style="color: green">sys.objects</span><font color="#000000"> </font><span style="color: blue">as</span><font color="#000000"> o <o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: gray">join</span><font color="#000000"> </font><span style="color: green">sys.partitions</span><font color="#000000"> </font><span style="color: blue">as</span><font color="#000000"> p </font><span style="color: blue">on</span><font color="#000000"> p</font><span style="color: gray">.</span><span style="color: fuchsia">object_id</span><font color="#000000"> </font><span style="color: gray">=</span><font color="#000000"> o</font><span style="color: gray">.</span><span style="color: fuchsia">object_id</span><font color="#000000"> <o:p></o:p></font></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span><font color="#000000"> </font></span><span style="color: blue">where</span><font color="#000000"> o</font><span style="color: gray">.</span><span style="color: blue">name</span><font color="#000000"> </font><span style="color: gray">=</span><font color="#000000"> </font><span style="color: red">&#8216;sysdesend&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p><font color="#000000" face="Times New Roman" size="3"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <font color="#000000" face="Times New Roman" size="3">Note that all these will return a raw count, w/o concerning the state of individual messages or conversations. E.g if retention is turned on a queue, the retained messages will also be counted. The partition rows count is not guaranteed to match the actual count returned by a COUNT(*) operator, but the situations where it diverges are really transient border cases unlikely to be hit. Besides, most of the times such counts are interesting for monitoring and management, when the exact result is not so much important as to whether to determine if certain thresholds are being passed and notifications have to be sent (e.g. when the count of messages in transmission_queue is growing).</font>
</p>