---
id: 1431
title: 'SQL Server 2008 R2 Express database size limit: 10GB'
date: 2010-04-28T10:11:11+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/04/28/788-revision-4/
permalink: /2010/04/28/788-revision-4/
---
The SQL Server 2008 R2 Express editions has increased the database size limit to 10Gb from the previous limit of 4Gb. This is great news for many developers, as the 4Gb limitation was by far the most difficult barrier preventing Express adoption. With today&#8217;s rate of generating data, the 4Gb limit was just plain small.

All the other limitations of SQL Server Express stay in place:

CPU
:   SQL Server Express only uses once CPU socket. It will use all cores and any Hyper-Threading logical processor in that socket though.

Memory
:   SQL Server Express limits the size of the data buffer pool to 1Gb.

Replication
:   SQL Server Express can only participate as a subscriber in a replication topology.

Service Broker
:   Two SQL Server Express instances cannot exchange Service Broker messages directly, the messages have to be routed through a higher level SKU.

SQL Agent
:   SQL Server Express does not have an Agent service and as such it cannot run Agent scheduled jobs.