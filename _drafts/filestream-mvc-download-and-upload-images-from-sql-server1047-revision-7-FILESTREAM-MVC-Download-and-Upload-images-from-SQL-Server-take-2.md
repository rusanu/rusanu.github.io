---
id: 1055
title: 'FILESTREAM MVC: Download and Upload images from SQL Server, take 2'
date: 2011-02-05T11:44:35+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/02/05/1047-revision-7/
permalink: /2011/02/05/1047-revision-7/
---
In a previous article I have shown how it is possible to use efficient streaming semantics when <a hre="http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/">Download and Upload images from SQL Server via ASP.Net MVC</a>. In this article I will go over an alternative approach that relies on the <a href="http://technet.microsoft.com/en-us/library/bb933993.aspx" target="_blank"><tt>FILESTREAM</tt></a> column types introduced in SQL Server 2008.

### What is FILESTREAM?

<tt>FILESTREAM</tt> storage is a new option available in SQL Server 2008 and later that allows for BLOB columns to be stored directly on the file system as individual files. As files, the data is accessible through the Win32 file access API like <a href="http://msdn.microsoft.com/en-us/library/aa365467%28v=vs.85%29.aspx" target="_blank"><tt>ReadFile</tt></a> and <a href="http://msdn.microsoft.com/en-us/library/aa365747%28v=vs.85%29.aspx" target="_blank"><tt>WriteFile</tt></a>. But at the same time the same data is available through the normal T-SQL operations like <tt>SELECT</tt> or <tt>UPDATE</tt>. Not only that, but the data _is_ contained logically in the database so it will be contained in a database backup, it is subject to ordinary transaction commit and rollback behavior, it is searched by SQL Server FullText indexes and it follows the normal SQL Server security access rules: if you are granted SELECT permission on the table, then you can open the file. There are some restrictions, eg. a database with FILESTREAM cannot be mirrored. For a full list of restrictions and limitations, see <a href="http://technet.microsoft.com/en-us/library/bb895334.aspx" target="_blank">Using FILESTREAM with Other SQL Server Features</a>. Note that SQL Server Express edition _does_ support FILESTREAM storage.

Another attribute of the FILESTREAM storage is the size limitation: normal BLOB column values have a maximum size of 2Gb. FILESTREAM columns are limited only by the volume size limit of the file system. 2Gb may seem like a large value, but consider that a media file like an HD movie stream can easily go up to a size of 5Gb.

## Using FILESTREAM

One way to use FILESTREAM columns is to treat them as ordinary BLOB values and manipulate them through T-SQL. The one restriction in place is that the effcient partial update syntax for BLOBs is not supported. That is, one cannot issue <tt>UPDATE table SET column.WRITE(...) WHERE ... </tt> on a FILESTREAM column. But where FILESTREAM storage begins to shine is when accessed through the system file API. This allows the application to efficiently read, write and seek in a large BLOB value, just as it would in a file. In fact, the application _does_ read, write and seek in a file :).

Native Win32 applications use a new API function <a href="http://msdn.microsoft.com/en-us/library/bb933972.aspx" target="_blank"><tt>OpenSqlFilestream</tt></a> that opens a <tt>HANDLE</tt> that can be then used with the file IO API functions. Managed applications use the new <a href="http://msdn.microsoft.com/en-us/library/system.data.sqltypes.sqlfilestream.aspx" target="_blank"><tt>SqlFileStream</tt></a> class that exposes a Stream based on the underlying FILESTREAM value. Both the native and the manged API require as input two special values, a <a href="http://technet.microsoft.com/en-us/library/bb895239.aspx" target="_blank"><tt>PathName</tt></a> and a <a href="http://technet.microsoft.com/en-us/library/bb934014.aspx" target="_blank"><tt>Transaction Context</tt></a> that must be obtained previously from the SQL Server using T-SQL.