---
id: 595
title: System pagefile size on machines with large RAM
date: 2009-11-22T15:20:45+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/11/22/594-revision/
permalink: /2009/11/22/594-revision/
---
Irrelevant of the size of the RAM, you still need a pagefile at least 1.5 times the amount of physical RAM. This is true even if you have a 1 TB RAM machine, you&#8217;ll need 1.5 TB pagefile on disk (sounds crazy, but is true)

When a process asks MEM_COMMIT memory via VirtualAlloc/VirtualAllocEx, the requested size needs to be reserved in the pagefile. This was true in the first Win NT system, and is still true today see <a href=" http://msdn.microsoft.com/en-us/library/ms810627.aspx" target="_blank">Managing Virtual Memory in Win32</a>

> When memory is committed, physical pages of memory are allocated **and space is reserved in a pagefile**. 

Bare some extreme odd cases, SQL Server will always ask for MEM_COMMIT pages. And given the fact that SQL uses a <a href="http://msdn.microsoft.com/en-us/library/ms178145.aspx" target="_blank">Dynamic Memory Management</a> policy that reserves upfront as much buffer pool as possible (reserves and _commits_ in terms of VAS), SQL Server will request at start up a huge reservation of space in the pagefile. If the pagefile is not properly sized errors 801/802 will start showing up in SQL&#8217;s ERRORLOG file and operations.

This always causes some confusion, as administrators erroneously assume that a large RAM eliminates the need for a pagefile. In truth the contrary happens, a large RAM increases the need for pagefile, just because of the inner workings of the Windows NT memory manager. Although reserved pagefile is, hopefully, never used, the problem of reserving such a huge pagefile file can be quite serious and needs to be accounted for during capacity planning.