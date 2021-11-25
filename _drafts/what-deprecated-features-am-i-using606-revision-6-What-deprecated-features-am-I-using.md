---
id: 2013
title: What deprecated features am I using?
date: 2010-01-12T12:37:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/01/12/606-revision-6/
permalink: /2010/01/12/606-revision-6/
---
<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">instance_name &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[Deprecated Feature]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">cntr_value &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[Frequency Used]&lt;br />
&lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Green">dm_os_performance_counters&lt;br />
&lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Fuchsia">object_name &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Red">'SQLServer:Deprecated Features'&lt;br />
&lt;/span>&lt;span style="color:Gray">and &lt;/span>&lt;span style="color:Black">cntr_value &lt;/span>&lt;span style="color:Gray">&gt; &lt;/span>&lt;span style="color:Black">0&lt;br />
&lt;/span>&lt;span style="color:Blue">order by &lt;/span>&lt;span style="color:Black">cntr_value &lt;/span>&lt;span style="color:Blue">desc&lt;/span>
&lt;/pre>
&lt;p></code>

Quick way to tell which deprecated feature are used on a running instance of SQL Server. The performance counters reset at each server start up, so the interogation is relevant only after the server was up for some time. This will not tell you where is the usage comming from, but will give you a very quick idea what deprecated features are used most frequently by your apps.

If the SQL Server is a named instance, you have to query the proper counter category: <tt><span style="color:Red">'MSSQL$<instancename>:Deprecated Features'</span></tt>