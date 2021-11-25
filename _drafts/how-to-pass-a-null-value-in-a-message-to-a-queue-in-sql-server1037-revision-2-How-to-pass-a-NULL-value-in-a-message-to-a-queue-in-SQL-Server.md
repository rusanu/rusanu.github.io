---
id: 1040
title: How to pass a NULL value in a message to a queue in SQL Server
date: 2011-01-15T09:56:05+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/01/15/1037-revision-2/
permalink: /2011/01/15/1037-revision-2/
---
The <a href="http://msdn.microsoft.com/en-us/library/ms188407.aspx" target="_blank">SEND</a> Transact-SQL verb does not allow to send a NULL message body, attempting to do so will result in error:

<pre>