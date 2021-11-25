---
id: 1998
title: SQL Server backup to URL
date: 2013-01-25T04:24:15+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/01/25/1706-revision-7/
permalink: /2013/01/25/1706-revision-7/
---
With the SQL Server 2012 SP1 CU2 release a new important feature was added: ability to [back up and restore a database straight from Azure Blob storage](http://msdn.microsoft.com/en-us/library/jj919148.aspx):

> This feature released in SQL Server 2012 SP1 CU2, enables SQL Server backup and restore directly to the Windows Azure Blob service. This feature can be used to backup SQL Server databases on an on-premises instance or an instance of SQL Server running a hosted environment such as Windows Azure Virtual Machine. Backup to cloud offers benefits such as availability, limitless geo-replicated off-site storage, and ease of migration of data to and from the cloud. 

The syntax for the new feature is straight forward. You must first create a credential for accessing your Azure Blob storage:

> SQL Server requires Windows Azure account name and access key authentication to be stored in a SQL Server Credential. This information is used to authenticate to the Windows Azure account when it performs backup or restore operations.


<code class="prettyprint lang-sql">
&lt;pre>
CREATE CREDENTIAL mycredential WITH IDENTITY = 'mystorageaccount'
       ,SECRET = '&lt;storage access key>' ;

BACKUP DATABASE AdventureWorks2012 
      TO URL = 'https://mystorageaccount.blob.core.windows.net/mycontainer/db.bak' 
      WITH CREDENTIAL = 'mycredential' 
     ,STATS = 5;

RESTORE DATABASE AdventureWorks2012 
     FROM URL = 'https://mystorageaccount.blob.core.windows.net/mycontainer/db.bak'
     WITH CREDENTIAL = 'mycredential'
&lt;/pre>
&lt;p></code>

Full backups, differential backups, log backups, filegroup backups, compressed backups are all supported. The only notable restrictions is that is not allowed to backup to two locations simultaneously. Here is the complete list of limitations:

>   * The maximum backup size supported is 1 TB.
>   * Backup to or restoring from the Windows Azure Blob storage service by using SQL Server Management Studio Backup or Restore wizard is not currently enabled
>   * Creating a logical device name is not supported. So adding URL as a backup device using sp_dumpdevice or through SQL Server Management Studio is not supported
>   * Appending to existing backup blobs is not supported. Backups to an existing Blob can only be overwritten by using the WITH FORMAT option
>   * Backup to multiple blobs in a single backup operation is not supported
>   * Specifying a block size with BACKUP is not supported.
>   * Specifying a block size for restores might be required in certain scenarios
>   * Specifying MAXTRANSFERSIZE is not supported.
>   * Specifying backupset options &#8211; RETAINDAYS and EXPIREDATE are not supported.
>   * SQL Server has a maximum limit of 259 characters for a backup device name. The BACKUP TO URL consumes 36 characters for the required elements used to specify the URL – ‘https://.blob.core.windows.net//.bak’, leaving 223 characters for account, container, and blob names put together.

I recommend going over [Backup and Restore to Azure Blob Best Practices](http://msdn.microsoft.com/en-us/library/jj919149.aspx).