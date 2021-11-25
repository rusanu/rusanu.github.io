---
id: 1366
title: Understanding Hash, Sort and Exchange Spill events
date: 2011-10-19T12:05:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/10/19/1324-revision-7/
permalink: /2011/10/19/1324-revision-7/
---
Certain SQL Server query execution operations are calibrated to perform best by using a (somewhat) large amount of memory as intermediate storage. The Query Optimizer will choose a plan and estimate the cost based on these operators using this memory scratch-pad, but the this is, of course, only an estimate. At execution the estimates may prove wrong and the plan must must continue despite not having enough memory. In such an event, these operators **spill** to disk. When a spill occurs, the memory scratch-pad is flushed into **tempdb** and more data is accommodated int he freed memory. When the data flushed to tempdb is needed again, is read from disk. Needless to say, spilling into tempdb is order of magnitude slower than using just the memory scratch-pad. Monitoring for spilling is specially important in ETL jobs, since these occurrences may cause the ETL job to stretch for many more minutes, sometimes even hours.

### [Hash Warning Event](http://technet.microsoft.com/en-us/library/ms190736.aspx)

> Hash recursion occurs when the build input does not fit into available memory, resulting in the split of input into multiple partitions that are processed separately. If any of these partitions still do not fit into available memory, it is split into subpartitions, which are also processed separately. This splitting process continues until each partition fits into available memory or until the maximum recursion level is reached (displayed in the IntegerData data column). 

This is one of the most common spills. Hash Join is a darling of the Query Optimizer, as it a very fats operator that can join two unsorted sources efficiently, requiring a single pass over each source. Not only that, but it can also be used for duplicate removal (eg. <tt>DISTINCT</tt> clause) and grouping aggregates. See [Understanding Hash Joins](http://technet.microsoft.com/en-us/library/ms189313.aspx) for more details.

Unfortunately it also requires a lot of memory. Hash spill occurrences are usually an indication of bad cardinality estimates that are in turn caused, most times, by missing or outdated statistics. The first action to do, when faced with Hash Join spill warnings, is to update (or create) stats on the columns involved. If the problem persists then you must resort to a different join type. Since the optimizer would had already picked a better join if it could, you need to look at the problem from a different angle: how could you help the optimizer to pick a different join, w/o _forcing_ it?

The optimizer would pick a LOOP if it could seek the inner side for a reasonable number of times, so perhaps you need an index on the inner side to allow a seek and/or an index on the outer side that would filter the produced output early so it can reduce the number of probes in the inner side. See [Understanding Nested Loops Joins](http://msdn.microsoft.com/en-us/library/ms191318.aspx) for more details.

The other alternative is a MERGE join, but merge requires the input on both sides to be sorted. Adding an index to each of the sources on both sides of the join that provides the order guarantee would likely woe the optimizer into using the MERGE join. See [Understanding Merge Joins](http://msdn.microsoft.com/en-us/library/ms190967.aspx) for more details.

### [Sort Warning Event](http://technet.microsoft.com/en-us/library/ms178041.aspx)

> The Sort Warnings event class indicates that sort operations do not fit into memory. This does not include sort operations involving the creation of indexes, only sort operations within a query (such as an ORDER BY clause used in a SELECT statement).

### [Exchange Spill Event](http://technet.microsoft.com/en-us/library/ms191514.aspx)

> The Exchange Spill event class indicates that communication buffers in a parallel query plan have been temporarily written to the tempdb database. This occurs rarely and only when a query plan has multiple range scans.