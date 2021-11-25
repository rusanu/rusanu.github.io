---
id: 971
title: 'This server supports version 662 and earlier&#8230;'
date: 2010-11-23T09:57:58+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/23/909-revision-6/
permalink: /2010/11/23/909-revision-6/
---
A new error started showing up in SQL Server 2008 SP2 installations:

> The database cannot be opened because it is version 661. This server supports version 662 and earlier. A downgrade path is not supported.

661 sure is earlier than 662, so what seems to be the problem? This error message is a bit misleading. SQL Server 2008 supports database version 655 and earlier. But with support for 15000 partitions in SQL Server 2008 SP2, databases enabled for 15000 partitions are upgraded to version 662. This upgrade is necessary to prevent an SQL Server 2008 **R2** instance from attaching a database that has more than 1000 partitions in it, since the code in R2 RTM does _not_ understand 15000 partitions and the effects would be unpredictable. So SQL Server 2008 SP2 does indeed support version 662, but it does **not** support version 661. This behavior is explained in the <a href="http://download.microsoft.com/download/B/E/1/BE1AABB3-6ED8-4C3C-AF91-448AB733B1AF/Support_for_15000_Partitions.docx" target="_blank">Support for 15000 Partitions.docx</a> document, although the database versions are not explictly called out.

So the error message above should be really read as:

> The database cannot be opened because it is version 661. This server supports versions 662, 655 and earlier than 655. A downgrade path is not supported

With this information the correct resolution can be achieved: the user is trying to attach a SQL Server 2008 R2 database (v. 661) to an SQL Server 2008 SP2 instance. This is not supported. User has to either upgrade the SQL Server 2008 SP2 instance to SQL Server 2008 R2, or it has to attach the database back to a R2 instance and copy out the data from the database into SQL Server 2008 instance database, eg. using the <a href="http://msdn.microsoft.com/en-us/library/ms140052.aspx" target="_blank">Import and Export Wizard</a>.