---
id: 1097
title: How to determine the database version of an MDF file
date: 2011-04-04T21:46:50+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/04/04/1095-revision-2/
permalink: /2011/04/04/1095-revision-2/
---
If you have an MDF file laying around and you don&#8217;t know what version is, how can you determine the version? The simple answer is that you can try to attach it to a SQL Server instance, but if you&#8217;re not careful the operation may end up upgrading the MDF file to that instance&#8217;s current version. Even if it doesn&#8217;t update it, it can still run recovery on the database and thus change the file. Is there a way to determine the version in a non destructive fashion?

Lets look into a database and dump a page. Not any page, but page 9 of the primary filegroup. This page happens to be the database boot page:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
dbcc traceon(3604)
dbcc page(1,1,9,3);
&lt;/pre>
&lt;p></code>

the important bits of information are these:

<pre>...
Memory Dump @0x00000000124FA060
0000000000000000:   0000a405 95029502 00000000 00000000 †..¤........... 
...
DBINFO @0x00000000124FA060
...
dbi_dbid = 1                         dbi_status = 65544                   dbi_nextid = 1723153184
dbi_dbname = master                  dbi_maxDbTimestamp = 4000            dbi_version = 661
dbi_createVersion = 661              dbi_ESVersion = 0                    
...
</pre>

So at offset 0x60 on the page (96 decimal, which we know is the size of the page header) there is a DBINFO structure. This structure contains the dbi_version 661, which is 0x295 in hex. In the DBCC PAGE output this would show as 9502, and we can see that there are two such values at offset 0x64 and 0x66. So the database version is stored in the two bytes at offset 0x64 on the 9th page of the primary filegroup. The 9 page will be at offset 0x12000 in the file (just take the page size, 8k, and multiply it by 9). therefore the version of the MDF will be the two bytes at offset 0x12064 in the file.

there are many ways you can read these bytes using your favourite hex file editor. Even Powershell can do it:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
PS C:\Windows\system32> get-content -Encoding Byte "...\foo.mdf" | select-object -skip 0x12064 -first 2
149
2
PS C:\Windows\system32> 2*256+149
661
&lt;/pre>
&lt;p></code>

The first Powershell line dumped the two bytes at offset 0x12064 in the file: 149 and 2 (Powersell displays the value in decimal encoding, which is pretty useless&#8230;). Convert the two bytes to a DWORD gives us the MDF file version: 661, which is the SQL Server 2008 R2 version.