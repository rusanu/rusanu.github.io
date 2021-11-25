---
id: 128
title: 'How long should I expect ALTER DATABSE &#8230; SET ENABLE_BROKER to run?'
date: 2008-01-03T16:20:23+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/01/03/74-revision/
permalink: /2008/01/03/74-revision/
---
 <span style="color: #333333; font-family: Verdana; font-size: 12px; line-height: 15px" class="Apple-style-span"></span> 

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  I&#8217;ve seen this question comming back again and again on forums and discussion groups. The ALTER DATABASE &#8230; SET ENABLE_BROKER seems to run forever and doesn&#8217;t complete. How long should one wait?
</p>

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  This statement completes imeadetly, but the problem is that is requires exclusive access to the database! Any connection that is using this database has a shared lock on it, even when idle, thus blocking the ALTER DATABASE from completing.
</p>

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  There is an easy trick for fix the problem: use the termination options of ALTER DATABASE:<font style="background-color: #ffffff" color="#0000ff">        </font>
</p>

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  <font style="background-color: #ffffff" color="#0000ff">ROLLBACK AFTER <em>integer</em> [ SECONDS ]         | ROLLBACK IMMEDIATE         | NO_WAIT</font>
</p>

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  -ROLLBACK <font face="Times New Roman">will close all existing sessions, rolling back any pending transaction.</font>
</p>

<p style="margin-top: 10px; margin-right: 0px; margin-bottom: 10px; margin-left: 0px">
  <font face="Courier New">&#8211; NO_WAIT</font> will terminate the ALTER DATABASE statement with error if other connections are blocking it.
</p>

 