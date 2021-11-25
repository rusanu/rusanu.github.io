---
id: 1385
title: 'What is an LSN: Log Sequence Number'
date: 2012-01-17T09:15:30+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/01/17/1381-revision-3/
permalink: /2012/01/17/1381-revision-3/
---
LSNs, or Log Sequence Numbers, are explained at <a href="http://msdn.microsoft.com/en-us/library/ms190411.aspx" target="_blank">Introduction to Log Sequence Numbers</a>:

> Every record in the SQL Server transaction log is uniquely identified by a log sequence number (LSN). LSNs are ordered such that if LSN2 is greater than LSN1, the change described by the log record referred to by LSN2 occurred after the change described by the log record LSN.