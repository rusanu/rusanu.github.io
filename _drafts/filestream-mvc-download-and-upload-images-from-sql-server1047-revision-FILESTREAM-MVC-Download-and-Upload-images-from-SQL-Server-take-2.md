---
id: 1048
title: 'FILESTREAM MVC: Download and Upload images from SQL Server, take 2'
date: 2011-02-04T17:14:57+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/02/04/1047-revision/
permalink: /2011/02/04/1047-revision/
---
In a previous article I have shown how it is possible to use efficient streaming semantics when <a hre="http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/">Download and Upload images from SQL Server via ASP.Net MVC</a>. In this article I will go over an alternative approach that relies on the <a href="http://technet.microsoft.com/en-us/library/bb933993.aspx" target="_blank"><tt>FILESTREAM</tt></a> column types introduced in SQL Server 2008.

### What is FILESTREAM?

<tt>FILESTREAM</tt> storage is a new option available in SQL Server 2008 and later that allows for BLOB columns to be stored directly on the file system as individual files. The data stored in these files is available through the normal T-SQL operations like <tt>SELECT</tt> or <tt>UPDATE</tt>, but at the same time the data is also available for the Win32 file access API like [<tt>ReadFile</tt>](http://msdn.microsoft.com/en-us/library/aa365467%28v=vs.85%29.aspx) and <a href=WriteFile