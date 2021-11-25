---
id: 1721
title: Why is SQL Server BULK INSERT faster than normal INSERT
date: 2013-02-14T00:31:21+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=1721
permalink: /?p=1721
categories:
  - Uncategorized
---
The discussion forums are full of advice to use [SqlBulkCopy](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlbulkcopy.aspx) class to insert rows faster. And experienced SQL Server ETL pipeline designers know that the preferred SSIS destimation is the [OLE DB table](http://msdn.microsoft.com/en-us/library/ms141237.aspx) with Fast Load. In fact these are various ways of saying &#8220;bulk insert&#8221;. Yet despite its popularity, I find bulk insert to be quite poorly understood.