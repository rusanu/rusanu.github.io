---
id: 1363
title: Understanding Hash, Sort and Exchange Spill events
date: 2011-10-19T11:42:13+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/10/19/1324-revision-4/
permalink: /2011/10/19/1324-revision-4/
---
Certain SQL Server query execution operations are calibrated to perform best by using a (somewhat) large amount of memory as intermediate storage. The Query Optimizer will choose a plan and estimate the cost based on these operators using this memory scratch-pad, but the this is, of course, only an estimate. At execution the estimates may prove wrong and the plan must must continue despite not having enough memory. In such an event, these operators **spill** to disk. When a spill occurs, the memory scratch-pad is flushed into **tempdb** and more data is accommodated int he freed memory. When the data flushed to tempdb is needed again, is read from disk. Needless to say, spilling into tempdb is order of magnitude slower than using just the memory scratch-pad. Monitoring for spilling is specially important in ETL jobs, since these occurrences may cause the ETL job to stretch for many more minutes, sometimes even hours.

### [Hash Warning Event](http://technet.microsoft.com/en-us/library/ms190736.aspx)

> Hash recursion occurs when the build input does not fit into available memory, resulting in the split of input into multiple partitions that are processed separately. If any of these partitions still do not fit into available memory, it is split into subpartitions, which are also processed separately. This splitting process continues until each partition fits into available memory or until the maximum recursion level is reached (displayed in the IntegerData data column). 

### [Sort Warning Event](http://technet.microsoft.com/en-us/library/ms178041.aspx)

> The Sort Warnings event class indicates that sort operations do not fit into memory. This does not include sort operations involving the creation of indexes, only sort operations within a query (such as an ORDER BY clause used in a SELECT statement).

### [Exchange Spill Event](http://technet.microsoft.com/en-us/library/ms191514.aspx)

> The Exchange Spill event class indicates that communication buffers in a parallel query plan have been temporarily written to the tempdb database. This occurs rarely and only when a query plan has multiple range scans.