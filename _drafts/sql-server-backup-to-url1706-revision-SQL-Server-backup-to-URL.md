---
id: 1707
title: SQL Server backup to URL
date: 2013-01-25T04:18:48+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/01/25/1706-revision/
permalink: /2013/01/25/1706-revision/
---
With the SQL Server 2012 SP1 CU2 release a new important feature was added: ability to [back up and restore a database straight from Azure Blob storage](http://msdn.microsoft.com/en-us/library/jj919148.aspx):

> This feature released in SQL Server 2012 SP1 CU2, enables SQL Server backup and restore directly to the Windows Azure Blob service. This feature can be used to backup SQL Server databases on an on-premises instance or an instance of SQL Server running a hosted environment such as Windows Azure Virtual Machine. Backup to cloud offers benefits such as availability, limitless geo-replicated off-site storage, and ease of migration of data to and from the cloud. 

The syntax for the new feature is straight forward. You must first create a credential for accessing your Azure Blob storage:

> SQL Server requires Windows Azure account name and access key authentication to be stored in a SQL Server Credential. This information is used to authenticate to the Windows Azure account when it performs backup or restore operations.

<code class="prettyprint lang-sql">&lt;br />
CREATE CREDENTIAL mycredential WITH IDENTITY = 'mystorageaccount'&lt;br />
       ,SECRET = '&lt;storage access key>' ;&lt;/p>
&lt;p>BACKUP DATABASE AdventureWorks2012&lt;br />
      TO URL = 'https://mystorageaccount.blob.core.windows.net/mycontainer/AdventureWorks2012.bak'&lt;br />
      WITH CREDENTIAL = 'mycredential'&lt;br />
     ,STATS = 5;&lt;/p>
&lt;p>RESTORE DATABASE AdventureWorks2012&lt;br />
     FROM URL = 'https://mystorageaccount.blob.core.windows.net/mycontainer/AdventureWorks2012.bak'&lt;br />
     WITH CREDENTIAL = 'mycredential'&lt;/p>
&lt;p></code>

Full backups, differential backups, log backups, filegroup backups, compressed backups are all supported. The only notable restrictions is that is not allowed to backup to two locations simultaneously