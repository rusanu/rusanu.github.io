---
id: 66
title: Is Service Broker in SQL Server 2008 compatible with the one in SQL Server 2005?
date: 2007-11-28T11:14:45+00:00
author: remus
layout: post
guid: /2007/11/28/is-service-broker-in-sql-server-2008-compatible-with-the-one-in-sql-server-2005/
permalink: /2007/11/28/is-service-broker-in-sql-server-2008-compatible-with-the-one-in-sql-server-2005/
categories:
  - Announcements
---
Now that the SQL Server 2008 CTP is no longer a virtual image but a true install and the quality of this CTP is near ship quality many of you are probably going to look at testing existing applications on SQL Server 2008 and perhaps start coding some pilot projects exploring the new features in SQL Server 2008. For the distributed applications Service Broker supports one question arrises immedeatly: is Service Broker in SQL Server 2008 compatible with SQL Server 2005?<!--more-->

In othe words, can a Service Broker service hosted in a SQL Server 2008 instance exchange messages with one hosted in SQL Server 2005 and vice-versa? The answer is **yes**. The wire communication protocol used by Service Broker is unchanged in SQL Server 2008 and the two services can exchange messages without any problem. In fact the version difference is completely transparent up to the point that a service _cannot tell_ if it communicates with a SQL Server 2005 instance or a SQL Server 2008 instance!

This will also be important later for upgrade scenarios: when dealing with tens or hundreds of deployed instances all communicating with each other (or perhaps all communicating with a central location) the fact that any of this instances can be upgraded individually and will continue to communicate completely transparent with all the other non-upgraded instances makes the global upgrade much easier as not global down time is required.

## Update 2014

As of current version of SQL Server 2014 this is still true. All SQL Server versions since SQL Server 2005 (2008, 2008R, 2012, 2014) talk the very same SSB protocol and are compatible with each other.