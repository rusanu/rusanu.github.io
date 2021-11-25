---
id: 2530
title: Identifying SqlConnection objects in a dump
date: 2017-09-23T00:56:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2017/09/23/2505-revision-v1/
permalink: /2017/09/23/2505-revision-v1/
---
I recently had to troubleshoot an ADO.Net connection pool exhaust issue. This problem may indicate a connection leak, but it can also be caused by an undersized connection pool. For analysis I used Windbg. For a quick introduction on how to use Windbg with managed code, see [here](https://docs.microsoft.com/en-us/dotnet/framework/tools/sos-dll-sos-debugging-extension). Once the problem was reproduced, I took a process dump, copied the dump to my laptop and started digging.

I started by finding a SqlConnection object in the dump. <tt>!DumpHeap</tt> can be used to dump _all_ objects in all heaps, which is rather verbose. By using the <tt>-type</tt> filter we can restrict the output to only the objects with type name that contain a given string, and I also restricted to display only the aggregated statistics (<tt>-stat</tt>) rather than each individual entry: 

<pre class="windbg"><code>
0:081&gt; !DumpHeap -type System.Data.SqlClient.SqlConnection  -stat
Statistics:
              MT    Count    TotalSize Class Name
00007fff50d4dd90        1           56 System.Data.SqlClient.SqlConnectionPoolGroupProviderInfo
00007fff50d4ed90        1           64 System.Data.SqlClient.SqlConnectionFactory
00007fff50b99cd0        1           80 System.Collections.Generic.Dictionary`2[[System.String, mscorlib],[System.Data.SqlClient.SqlConnectionStringBuilder+Keywords, System.Data]]
00007fff50d4e730        1          216 System.Data.SqlClient.SqlConnectionString
00007fff51348288        1         1440 System.Collections.Generic.Dictionary`2+Entry[[System.String, mscorlib],[System.Data.SqlClient.SqlConnectionStringBuilder+Keywords, System.Data]][]
00007fff5135b5d0       51         1632 System.Data.SqlClient.SqlConnection+&lt;>c__DisplayClass152_0`1[[System.Int32, mscorlib]]
00007fff091c6940      133         4256 System.Data.SqlClient.SqlConnection+&lt;>c__DisplayClass152_0`1[[System.Data.SqlClient.SqlDataReader, System.Data]]
00007fff50d4dd20      100         4800 System.Data.SqlClient.SqlConnectionTimeoutErrorInternal
00007fff50d4cca0      133         6384 System.Data.SqlClient.SqlConnectionPoolKey
00007fff50d507d8      100         9600 System.Data.SqlClient.SqlConnectionTimeoutPhaseDuration[]
00007fff51358b10      377        12064 System.Data.SqlClient.SqlConnection+&lt;>c__DisplayClass114_0
00007fff50d4e838      700        16800 System.Data.SqlClient.SqlConnectionTimeoutPhaseDuration
00007fff50d49ee8      414        89424 System.Data.SqlClient.SqlConnection
Total 2013 objects
</code></pre>

From this command I am interested in only one thing, the method table address for the actual <tt>System.Data.SqlClient.SqlConnection</tt> type. With this I can restrict the <tt>!DumpHeap</tt> output to only this type objects, using the <tt>-mt</tt> argument. Notice that there are 414 SqlConnection instances in the dump, although we have a default connection pool size of 100. Lets see them:

<pre class="windbg"><code>
0:081&gt; !DumpHeap -mt 00007fff50d49ee8
         Address               MT     Size
000000de8bc9e270 00007fff50d49ee8      216     
000000de8bcf3890 00007fff50d49ee8      216   
...
000000e20e475910 00007fff50d49ee8      216     
000000e20e4dbe18 00007fff50d49ee8      216     
000000e20e5fb6f0 00007fff50d49ee8      216 
</code></pre>

If you enabled [the DML preference](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-markup-language-commands) then the output contains hot links to dumping each object with a simple click. This is just a shortcut and you can see the actual Windbg command run by the shortcut click. Lets dump the first object:

<pre class="windbg"><code>
0:081&gt; !DumpObj /d 000000de8bc9e270
Name:        System.Data.SqlClient.SqlConnection
MethodTable: 00007fff50d49ee8
EEClass:     00007fff50b9ed10
Size:        216(0xd8) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff6069e068  4000577        8        System.Object  0 instance 0000000000000000 __identity
00007fff5f68daa8  4002882       10 ...ponentModel.ISite  0 instance 0000000000000000 site
00007fff5f68f3c0  4002883       18 ....EventHandlerList  0 instance 0000000000000000 events
...
00007fff50d4e968  4000efc       98 ...ConnectionOptions  0 instance 0000000000000000 _userConnectionOptions
00007fff50d4e5d0  4000efd       a0 ...nnectionPoolGroup  0 instance 0000000000000000 _poolGroup
00007fff50d4c080  4000efe       a8 ...onnectionInternal  0 instance 000000e10b4f8418 _innerConnection
00007fff606a03d0  4000eff       b4         System.Int32  1 instance                5 _closeCount
00007fff606a03d0  4000f01       b8         System.Int32  1 instance           368526 ObjectID
...
</code></pre>

Those used to native debugging and the eternal chase after symbol files will appreciate the fact that managed data types are self describing even on retail. I removed some clutter, the information I&#8217;m after is the <tt>_innerConnection</tt>. The SqlConnection is just a wrapper. Lets look at it:

<pre class="windbg"><code>
0:081> !DumpObj /d 000000e10b4f8418 
Name:        System.Data.ProviderBase.DbConnectionClosedPreviouslyOpened
MethodTable: 00007fff50d32a18
EEClass:     00007fff50bc7ca0
Size:        96(0x60) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff606a03d0  4001a5d       38         System.Int32  1 instance                6 _objectID
...
00007fff45d81858  4001a6d       30 ...tions.Transaction  0 instance 0000000000000000 _enlistedTransactionOriginal
</code></pre>

OK, so this inner connection is of the type <tt>DbConnectionClosedPreviouslyOpened</tt>, and I can make an educated guess that this is not relevant for my investigation. Lets look instead at the last SqlConnection object from my list of 414 instances, and its <tt>_innerConnection</tt>:

<pre class="windbg"><code>
0:08&gt;> !DumpObj 000000e20e5fb6f0
Name:        System.Data.SqlClient.SqlConnection
MethodTable: 00007fff50d49ee8
EEClass:     00007fff50b9ed10
Size:        216(0xd8) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff6069e068  4000577        8        System.Object  0 instance 0000000000000000 __identity
...
00007fff50d4e5d0  4000efd       a0 ...nnectionPoolGroup  0 instance 000000e08b7e0170 _poolGroup
00007fff50d4c080  4000efe       a8 ...onnectionInternal  0 instance 000000df8c001ab0 _innerConnection
00007fff606a03d0  4000eff       b4         System.Int32  1 instance                1 _closeCount
...

0:081&gt; !DumpObj /d 000000df8c001ab0
Name:        System.Data.SqlClient.SqlInternalConnectionTds
MethodTable: 00007fff50d4c528
EEClass:     00007fff50b9f1b0
Size:        400(0x190) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff606a03d0  4001a5d       38         System.Int32  1 instance               95 _objectID
00007fff6068d6f8  4001a60       44       System.Boolean  1 instance                0 _allowSetConnectionString
...
00007fff6069da88  400109d      130        System.String  0 instance 0000000000000000 _routingDestination
00007fff60699ca0  4001089      9b8      System.TimeSpan  1   shared           static _dbAuthenticationContext
</code></pre>

This time the inner connection is of type [<tt>System.Data.SqlClient.SqlInternalConnectionTds</tt>](http://referencesource.microsoft.com/#System.Data/System/Data/SqlClient/SqlInternalConnectionTds.cs), which is more interesting. Before you object that how are _you_ going to find the SqlConnection objects that have inner connection of type <tt>SqlInternalConnectionTds</tt>, short of searching through all instances, let me say that you don&#8217;t have to. All I wanted so far was to do an introduction to the <tt>SqlInternalConnectionTds</tt> class, which is the one we&#8217;re really interested in. I could have started directly with it and ask you to take my word for it,  
but I wanted you to see how I came to find this class. So lets look at all the instances of this class, using <tt>!DumpHeap -mt ...</tt> with this class method table address:

<pre class="windbg"><code>
0:081&gt; !DumpHeap -mt 00007fff50d4c528
         Address               MT     Size
000000de8bd10908 00007fff50d4c528      400     
000000de8bd175c8 00007fff50d4c528      400     
000000de8bd2a200 00007fff50d4c528      400     
000000de8bd32a80 00007fff50d4c528      400  
...
000000e20c2d20a0 00007fff50d4c528      400     
000000e20c2f89a0 00007fff50d4c528      400     

Statistics:
              MT    Count    TotalSize Class Name
00007fff50d4c528      100        40000 System.Data.SqlClient.SqlInternalConnectionTds
</code></pre>

There are exactly 100 instances in memory, so this really is the class we&#8217;re after. It represents an open SqlConnection. So we have a way of finding all active, open SQL connections in memory. What can we do with this information? The <tt>SqlInternalConnectionTds</tt> type has the <tt>_parser</tt> field which is an [<tt>TdsParser</tt>](http://referencesource.microsoft.com/#System.Data/System/Data/SqlClient/TdsParser.cs) instance, and this type has the <tt>_physicalStateObj</tt> field which is a [<tt>TdsParserStateObject</tt>](http://referencesource.microsoft.com/#System.Data/System/Data/SqlClient/TdsParserStateObject.cs) instance, and this type has the <tt>_outBuff</tt> field of type <tt>Byte[]</tt> and I will make an educated guess that this is the _last packet sent to the SQL server on this connection_:

<pre class="windbg"><code>
0:117&gt; !DumpObj /d 000000de8bd10908 
Name:        System.Data.SqlClient.SqlInternalConnectionTds
MethodTable: 00007fff50d4c528
EEClass:     00007fff50b9f1b0
Size:        400(0x190) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff606a03d0  4001a5d       38         System.Int32  1 instance              171 _objectID
...
00007fff50d4dd90  4001076       90 ...GroupProviderInfo  0 instance 000000e08b7e0808 _poolGroupProviderInfo
00007fff50d4d580  4001077       98 ...lClient.TdsParser  0 instance 000000de8bd115b0 _parser
...
00007fff60699b20  400109c      178          System.Guid  1 instance 000000de8bd10a80 _originalClientConnectionId
00007fff6069da88  400109d      130        System.String  0 instance 0000000000000000 _routingDestination
00007fff60699ca0  4001089      9b8      System.TimeSpan  1   shared           static _dbAuthenticationContextLockedRefreshTimeSpan
...

0:117&gt; !DumpObj /d 000000de8bd115b0 
Name:        System.Data.SqlClient.TdsParser
MethodTable: 00007fff50d4d580
EEClass:     00007fff50b9f470
Size:        160(0xa0) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff606a03d0  4001336       70         System.Int32  1 instance              166 _objectID
00007fff50d4c790  4001338        8 ...ParserStateObject  0 instance 000000de8bd11650 _physicalStateObj
00007fff50d4c790  4001339       10 ...ParserStateObject  0 instance 0000000000000000 _pMarsPhysicalConObj
...
00007fff606a03d0  4001335      8e0         System.Int32  1   shared           static _objectTypeCount
...

0:117&gt; !DumpObj /d 000000de8bd11650 
Name:        System.Data.SqlClient.TdsParserStateObject
MethodTable: 00007fff50d4c790
EEClass:     00007fff50bd7608
Size:        472(0x1d8) bytes
File:        C:\Windows\Microsoft.Net\assembly\GAC_64\System.Data\v4.0_4.0.0.0__b77a5c561934e089\System.Data.dll
Fields:
              MT    Field   Offset                 Type VT     Attr            Value Name
00007fff606a03d0  400142f      150         System.Int32  1 instance              166 _objectID
00007fff50d4d580  4001430        8 ...lClient.TdsParser  0 instance 000000de8bd115b0 _parser
00007fff50d4ebf0  4001431       10 ...lClient.SNIHandle  0 instance 000000de8bd11f20 _sessionHandle
00007fff60699a20  4001432       18 System.WeakReference  0 instance 000000de8bd11828 _owner
00007fff50d41dc0  4001433       20 ...eader+SharedState  0 instance 0000000000000000 _readerState
00007fff606a03d0  4001434      154         System.Int32  1 instance                0 _activateCount
00007fff606a03d0  4001435      158         System.Int32  1 instance                8 _inputHeaderLen
00007fff606a03d0  4001436      15c         System.Int32  1 instance                8 _outputHeaderLen
00007fff606a27a8  4001437       28        System.Byte[]  0 instance 000000de8bd140c8 _outBuff
00007fff606a03d0  4001438      160         System.Int32  1 instance                8 _outBytesUsed
00007fff606a27a8  4001439       30        System.Byte[]  0 instance 000000de8bd12170 _inBuff
00007fff606a03d0  400143a      164         System.Int32  1 instance               41 _inBytesUsed
00007fff606a03d0  400143b      168         System.Int32  1 instance               41 _inBytesRead
...

0:117&gt; !DumpObj /d 000000de8bd140c8 
Name:        System.Byte[]
MethodTable: 00007fff606a27a8
EEClass:     00007fff600b2348
Size:        8024(0x1f58) bytes
Array:       Rank 1, Number of elements 8000, Type Byte (Print Array)
Content:     ...".............._...E......................4..I.N.S.E.R.T. .[.d.b.o.]...[
Fields:
</code></pre>

So it looks like we did found the last command sent on this connection to the SQL Server, but lets dig a bit deeper. Lets dump the <tt>Byte[]</tt> memory:

<pre class="windbg"><code>
0:117&gt; db 000000de8bd140c8
000000de`8bd140c8  a8 27 6a 60 ff 7f 00 00-40 1f 00 00 00 00 00 00  .'j`....@.......
000000de`8bd140d8  0e 11 00 22 00 00 01 00-16 00 00 00 12 00 00 00  ..."............
000000de`8bd140e8  02 00 5f 0a 00 00 45 00-00 00 01 00 00 00 07 00  .._...E.........
000000de`8bd140f8  00 00 02 00 00 00 e7 8c-04 09 04 d0 00 34 8c 04  .............4..
000000de`8bd14108  49 00 4e 00 53 00 45 00-52 00 54 00 20 00 5b 00  I.N.S.E.R.T. .[.
</code></pre>

The first 16 bytes are for the <tt>Byte[]</tt> internal fields, and the actual array starts at 000000de\`8bd140d8 (ie. the second line). To understand this packet, we need to understand the TDS protocol. Luckily the protocol is well documented at [[MS-TDS]: Tabular Data Stream Protocol](https://msdn.microsoft.com/en-us/library/dd304523.aspx). Every packet starts with a [packet header](https://msdn.microsoft.com/en-us/library/dd340948.aspx) and the first byte is the [packet type](https://msdn.microsoft.com/en-us/library/dd304214.aspx).  
In our case the packet type is <tt>0e</tt>, which is described in the previous link as a [Transaction manager request](https://msdn.microsoft.com/en-us/library/dd304528.aspx). So the last operation done with this connection was a request to enlist in a distributed transaction. This behavior is typical of [SqlConnection enlisting themselves in lightweight DTC because of a TransactionScope](https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/system-transactions-integration-with-sql-server).

With all the knowledge we have so far we can use [Windbg script](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/using-script-files) to dump all the live connections and information about the last packet they sent to the server:

<pre class="windbg"><code>

&lt;/object>
</code></pre>

I reckon Windbg scripts are a bit arcane in syntax. <tt>poi</tt> is used to read a pointer value from an address. <tt>by</tt> reads a single low-order byte from an address. And the magic values like 98, 8 and 28 are the offsets of the relevant fields in the object instance:

  * _parser at offset 98: 
        00007fff50d4d580  4001077       98 ...lClient.TdsParser  0 instance 000000de8bd115b0 _parser

  * _physicalStateObj at offset 8: 
        0007fff50d4c790  4001338        8 ...ParserStateObject  0 instance 000000de8bd11650 _physicalStateObj

  * _outBuff at offset 28: 
        00007fff606a27a8  4001437       28        System.Byte[]  0 instance 000000de8bd140c8 _outBuff

The script uses <tt>.foreach(obj {!DumpHeap ... -short})</tt> to iterate through each instance of type <tt>SqlInternalConnectionTds</tt> in memory, and then for each instance it outputs Debug Markup Language (DML) links to output, in a tabular form, the <tt>SqlInternalConnectionTds</tt> instance, the <tt>_parser</tt>, the <tt>_physicalStateObj</tt> and the <tt>_outBuff</tt>. It then prints the last sent packet type:

  * **BTC** is a SQL Batch
  * **RCP** is a SQL procedure call (this includes any SqlCommand batch that has parameters)
  * **BLK** is a BULK INSERT operation
  * **ATN** is an attention (a request to cancel an executing command)
  * **XML** is a request to enlist in a DTC

For RPC and Batch requests the script also dumps out the actual SQL command in the request. Note that this is the _last_ packet sent, and if the SQL text was bigger than one packet then the last packet will contain only the ending of the actual command sent. Here is the output from my actual dump I was analyzing:

<pre class="windbg"><code>
0:117&gt; $$&gt;&lt;c:\temp\dump_all_out_buffs.txt
1 000000de8bd10908 000000de8bd115b0 000000de8bd11650 000000de8bd140c8 XMR
2 000000de8bd175c8 000000de8bd18258 000000de8bd182f8 000000df0bc17180 XMR
3 000000de8bd2a200 000000de8bd2ae90 000000de8bd2af30 000000de8bd2d9a8 XMR
4 000000de8bd32a80 000000de8bd33710 000000de8bd337b0 000000de8bd4e5f8 XMR
5 000000de8bdba120 000000de8bdc1410 000000de8bdc14b0 000000de8bdc3f50 XMR
6 000000de8be0cbb8 000000de8be0d848 000000de8be0d8e8 000000de8be10360 RPC Time], [Var_24].[LastModifierUserId] AS [LastModifierUserId], [Var_24].[CreationTime] AS [CreationTime], [Var_24]...
...
15 000000de8c17ee78 000000de8c19e818 000000de8c19e8b8 000000de8c1a1670 XMR
16 000000de8c18ddf8 000000de8c18ea88 000000de8c274270 000000de8c27e908 RPC SELECT [Extent2].[Name] AS [Name] FROM   (SELECT [Var_20].[UserId] AS [UserId], [Var_20].[RoleId]...
...
90 000000e20be179b8 000000e20be18648 000000e20be186e8 000000e00baecb50 RPC UPDATE [dbo].[Sessions] SET ...
91 000000e20be97138 000000e20be97dc8 000000e20be97e68 000000e20bb526f8 RPC UPDATE [dbo].[Sessions] SET ... 
...
99 000000e20c2d20a0 000000e20c2d2d48 000000e20c2d2de8 000000e20c2ed5b8 XMR
100 000000e20c2f89a0 000000e20c2fe4c0 000000e20c2fe560 000000e20c302620 XMR
</code></pre>

You can see all 100 connections in the connection pool, what was the last packet type sent to the server by each connection, and for the applicable cases, the last SQL command executed on that connection.

As an alternative to the cumbersome Windbg script syntax you can try [DbgScript](https://github.com/alexbudmsft/dbgscript), which allows you to use python, ruby, LUA or other scripting languages to control Windbg.