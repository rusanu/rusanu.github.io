---
id: 349
title: A fix for error Cannot find the remote service SqlQueryNotificationService-GUID
date: 2009-04-18T00:51:50+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/04/18/345-revision-4/
permalink: /2009/04/18/345-revision-4/
---
Sometimes your ERRORLOG is peppered with messages complaining about the service SqlQueryNotificationService-<guid> not existing or query notification dialogs being closed because they received an error message with the text <tt>Remote service has been dropped</tt>. I have blogged about this problem before: <a href="http://rusanu.com/2007/11/10/when-it-rains-it-pours/" target="_blank">http://rusanu.com/2007/11/10/when-it-rains-it-pours/</a>. Unfortunately this problem was not under your control as an administrator nor as a developer. It is caused by the way the SqlDependency component of ADO.Net deploys the temporary service, queue and procedure needed for its functioning. The problem could be caused by your application calling SqlDependency.Stop inadvertently but also by simple timing problems:<a href="http://rusanu.com/2008/01/04/sqldependencyonchange-callback-timing/" target="_blank">http://rusanu.com/2008/01/04/sqldependencyonchange-callback-timing/</a>.

Good news: Microsoft has shipped a fix for this issue: <a href="http://support.microsoft.com/kb/958006" target="_blank">http://support.microsoft.com/kb/958006</a>. According to the knowledge base article you need to install the following Cumulative Update depending on your current version of SQL Server deployed:

  * For SQL Server 2005 SP2 you need <a href="http://support.microsoft.com/kb/956854/LN/" target="_blank">CU 10</a>.
  * For SQL Server 2005 SP3 you need <a href="http://support.microsoft.com/kb/959195/LN/" target="_blank">CU 1</a>.
  * for SQL Server 2008 you need <a href="http://support.microsoft.com/kb/958186/" target="_blank">CU 2</a>.

If you have SQL Server 2008 SP1 deployed you do not need to install any fix because the issue is fixed in SP1 for 2008.