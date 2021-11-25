---
id: 2012
title: What deprecated features am I using?
date: 2013-08-02T00:50:01+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/02/606-autosave/
permalink: /2013/08/02/606-autosave/
---
<pre><code class="prettyprint lang-sq linenums">
select instance_name as [Deprecated Feature] , 
	cntr_value as [Frequency Used] 
	from sys.dm_os_performance_counters 
	where object_name = 'SQLServer:Deprecated Features' 
	and cntr_value > 0 
	order by cntr_value desc;
</code>
</pre>

Quick way to tell which deprecated feature are used on a running instance of SQL Server. The performance counters reset at each server start up, so the interogation is relevant only after the server was up for some time. This will not tell you where is the usage comming from, but will give you a very quick idea what deprecated features are used most frequently by your apps.

If the SQL Server is a named instance, you have to query the proper counter category: <tt><span style="color:Red">'MSSQL$<instancename>:Deprecated Features'</span></tt>