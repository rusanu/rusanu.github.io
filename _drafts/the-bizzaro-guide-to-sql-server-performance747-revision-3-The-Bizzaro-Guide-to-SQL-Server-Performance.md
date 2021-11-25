---
id: 750
title: The Bizzaro Guide to SQL Server Performance
date: 2010-03-31T21:28:16+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/31/747-revision-3/
permalink: /2010/03/31/747-revision-3/
---
Some say performance troubleshooting is a difficult science that blends just the right amount of patience, knowledge and experience. But I say forget all that, a few bullet points can get you a long way in fixing any problem you encounter. Is more important to find a google SEO friendly result that gives simplistic advice. Most importantly, good advice never contains the words &#8216;It depends&#8217;. Without further ado, here is my bulletproof SQL Server optimization guide:

  * **Always trust your gut feeling.** Avoid doing costly and unnecessary measurements. They may lead down the treacherous path of the scientific method. A gut feeling is always easier to explain and this improves communication. Measurements require use of complicated notions not everybody understands, so they lead to conflicts in the team.
  * **High CPU utilization is caused by index fragmentation.** Because the distance between database pages increases, the processor needs more cycles to reference the pages in the buffer pool.
  * **Low CPU utilization is caused by index fragmentation.** As the index fragments get smaller, they fit better into the processor L2 cache and this results in fewer cycles needed to access the row slots in the page. Because the data is in the cache the processor idles next cycles, resulting in low CPU utilization.
  * **High Avg. Disk Sec. per Transfer is caused by index fragmentation.** When indexes are fragmented the disk controller has to reorder the IO scatter-gather requests to put them in descending order. Needles to say, this operation increases the transfer times in geometric progression, because all the commercial disk controllers use bubble sort for this operation.
  * **High memory consumption is caused by index fragmentation.** This is fairly trivial and well known, but I&#8217;ll repeat it here: as the number of index fragments increases more pointers are needed to keep track of each fragment. Pointers are stored in virtual memory and virtual memory is very large, and this causes high memory consumption.
  * **Syntax errors are caused by index fragmentation.** Because the syntax is verified using the metadata catalogs, high fragmentation in the database can leave gaps in the syntax. This is turn causes the parser to generate syntax errors on perfectly valid statements like SECLET and UPTADE.
  * **Covering indexes can lead to index fragmentation.** Covering indexes are the indexes used by the query optimizer to cover itself in case the plan has execution faults. Because they are so often read they wear off and start to fragment.
  * **Index fragmentation can be resolved by shrinking the database**. As the data pages are squeezed tighter during the shrinking, they naturally realign themselves in the correct order.

There you have it, the simplest troubleshooting guide. Since most performance problems are caused by index fragmentation, all you have to do is shrink the database to force the pages to re-align correctly, and this will resolve the performance problem.