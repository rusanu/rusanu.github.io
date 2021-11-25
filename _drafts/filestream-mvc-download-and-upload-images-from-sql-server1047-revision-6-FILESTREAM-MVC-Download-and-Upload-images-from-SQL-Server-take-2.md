---
id: 1054
title: 'FILESTREAM MVC: Download and Upload images from SQL Server, take 2'
date: 2011-02-05T11:22:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/02/05/1047-revision-6/
permalink: /2011/02/05/1047-revision-6/
---
In a previous article I have shown how it is possible to use efficient streaming semantics when <a hre="http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/">Download and Upload images from SQL Server via ASP.Net MVC</a>. In this article I will go over an alternative approach that relies on the <a href="http://technet.microsoft.com/en-us/library/bb933993.aspx" target="_blank"><tt>FILESTREAM</tt></a> column types introduced in SQL Server 2008.

### What is FILESTREAM?

<tt>FILESTREAM</tt> storage is a new option available in SQL Server 2008 and later that allows for BLOB columns to be stored directly on the file system as individual files. As files, the data is accessible through the Win32 file access API like <a href="http://msdn.microsoft.com/en-us/library/aa365467%28v=vs.85%29.aspx" target="_blank"><tt>ReadFile</tt></a> and <a href="http://msdn.microsoft.com/en-us/library/aa365747%28v=vs.85%29.aspx" target="_blank"><tt>WriteFile</tt></a>. But at the same time the same data is available through the normal T-SQL operations like <tt>SELECT</tt> or <tt>UPDATE</tt>. Not only that, but the data _is_ contained logically in the database so it will be contained in a database backup, it is subject to ordinary transaction commit and rollback behavior, it is searched by SQL Server FullText indexes and it follows the normal SQL Server security access rules: if you are granted SELECT permission on the table, then you can open the file. There are some restrictions, eg. a database with FILESTREAM cannot be mirrored. For a full list of restrictions and limitations, see <a href="http://technet.microsoft.com/en-us/library/bb895334.aspx" target="_blank">Using FILESTREAM with Other SQL Server Features</a>. Note that SQL Server Express editions do support FILESTREAM storage.