---
id: 1153
title: SQL Server Database File Structure
date: 2011-06-13T23:26:59+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/06/13/1107-revision-5/
permalink: /2011/06/13/1107-revision-5/
---
In this article I want to talk about the low level organization of SQL Server database files. How is a database file structures, what are the Pages, Extents and Allocation Chains everyone is talking about. At a high level we all know that a SQL Server database contains tables and indexes, but to understand how these tables and indexes are layout on disk, I feel there is a need to explain the low level of the physical structure.

# Pages

&#8216;

All SQL Server database files are split into &#8220;pages&#8221;. A page is nothing more than an 8Kb (8192 bytes) length portion of the file. The bytes 0 to 8191 are &#8220;page 1&#8221;, the bytes 8192 to 16383 are &#8220;page 2&#8221;, bytes 16384 to 24575 are &#8220;page 3&#8221; and so on and so forth. So when we say &#8220;page 23&#8221; we simply mean the portion of the file between byte 188416 and byte 196607. SQL Server will uses &#8220;page&#8221; as the primitive structure upon which higher level abstractions are built. SQL Server also always reads and writes entire pages to and from disk.