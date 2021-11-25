---
id: 1371
title: Understanding Hash, Sort and Exchange Spill events
date: 2011-10-19T12:27:11+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/10/19/1324-revision-12/
permalink: /2011/10/19/1324-revision-12/
---
Certain SQL Server query execution operations are calibrated to perform best by using a (somewhat) large amount of memory as intermediate storage. The Query Optimizer will choose a plan and estimate the cost based on these operators using this memory scratch-pad, but the this is, of course, only an estimate. At execution the estimates may prove wrong and the plan must must continue despite not having enough memory. In such an event, these operators **spill** to disk. When a spill occurs, the memory scratch-pad is flushed into **tempdb** and more data is accommodated in the (now) free memory. When the data flushed to tempdb is needed again, is read from disk. Needless to say, spilling into tempdb is order of magnitude slower than using just the memory scratch-pad. Monitoring for spilling is specially important in ETL jobs, since these occurrences may cause the ETL job to stretch for many more minutes, sometimes even hours. For an exhaustive discussion of ETL, including some references to spills, see [The Data Loading Performance Guide](http://msdn.microsoft.com/en-us/library/dd425070%28v=sql.100%29.aspx).

### [Hash Warning Event](http://technet.microsoft.com/en-us/library/ms190736.aspx)

> Hash recursion occurs when the build input does not fit into available memory, resulting in the split of input into multiple partitions that are processed separately. If any of these partitions still do not fit into available memory, it is split into subpartitions, which are also processed separately. This splitting process continues until each partition fits into available memory or until the maximum recursion level is reached (displayed in the IntegerData data column). 

This is one of the most common spills. Hash Join is a darling of the Query Optimizer, as it a very fast operator that can join two unsorted sources efficiently, requiring a single pass over each source. Not only that, but it can also be used for duplicate removal (eg. <tt>DISTINCT</tt> clause) and grouping aggregates. See [Understanding Hash Joins](http://technet.microsoft.com/en-us/library/ms189313.aspx) for more details.

Unfortunately it also requires a lot of memory. Hash spill occurrences are usually an indication of bad cardinality estimates that are in turn caused, most times, by missing or outdated statistics. The first action to do, when faced with Hash Join spill warnings, is to update (or create) stats on the columns involved. If the problem persists then you must resort to a different join type. Since the optimizer would had already picked a better join if it could, you need to look at the problem from a different angle: how could you help the optimizer to pick a different join, w/o _forcing_ it?

The optimizer would pick a LOOP if it could seek the inner side for a reasonable number of times, so perhaps you need an index on the inner side to allow a seek and/or an index on the outer side that would filter the produced output early so it can reduce the number of probes in the inner side. See [Understanding Nested Loops Joins](http://msdn.microsoft.com/en-us/library/ms191318.aspx) for more details.

The other alternative is a MERGE join, but merge requires the input on both sides to be sorted. Adding an index to each of the sources on both sides of the join that provides the order guarantee would likely woe the optimizer into using the MERGE join. See [Understanding Merge Joins](http://msdn.microsoft.com/en-us/library/ms190967.aspx) for more details.

### [Sort Warning Event](http://technet.microsoft.com/en-us/library/ms178041.aspx)

> The Sort Warnings event class indicates that sort operations do not fit into memory. This does not include sort operations involving the creation of indexes, only sort operations within a query (such as an ORDER BY clause used in a SELECT statement).

If a query requires an order guarantee (eg. it has an ORDER BY clause, or it projects a function like ROW_NUMBER) and there is no index to guarantee the order then there is little choice to have: the execution must sort the input before proceeding. If the input is small then the sort occurs in memory and is very cheap, but in this case no spill warning would occur. The Query Optimizer is usually smart to postpone the sorting until all filtering is complete, so if it still spills it usually means there is no more filtering that can be applied. The solution when faced with Sort spills is usually to add a covering index that provides the desired order.

Another case of Sort spill warning is when the Query Optimizer goes creative and adds a sort into a plan that technically does not require an order guarantee. Such events are though very rare and esoteric. The action to take really depends from case to case.

### [Exchange Spill Event](http://technet.microsoft.com/en-us/library/ms191514.aspx)

> The Exchange Spill event class indicates that communication buffers in a parallel query plan have been temporarily written to the tempdb database. This occurs rarely and only when a query plan has multiple range scans. 

You&#8217;ll probably never going to hit this problem. If you do, I recommend you go over the recommendations in the licked article: [Exchange Spill Event](http://technet.microsoft.com/en-us/library/ms191514.aspx)

### Conclusion

Technically there is one more spill class: the spool spill. Since spools are \*meant\* to spill, the presence of a spool spill is usually less of a concern.

The purpose of this article is to show that there is ample documentation available on MSDN regarding these spill events. The tempdb spills are easily detectable and reasonably explained in the product documentation. Presence of spills may indicate potential performance problems as a spill involves disk reads and writes and is many times slower than the corresponding in-memory-only operation. They also add overload to tempdb and may cause contention.