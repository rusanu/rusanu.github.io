---
id: 267
title: CLR Memory Leak
date: 2009-01-19T13:23:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/01/19/264-revision-3/
permalink: /2009/01/19/264-revision-3/
---
On a recent client engagement I had to investigate what appeared a to be a memory leak in a manager application. The program was running for about a week now and it appeared to slowly degrade in performance over time. Although it appeared healthy at only about 245 MB of memory used, I decided to investigate. The fastest way in my opinion to down leaks in a running environment is to attach Windbg and use SOS:

  * Download the appropriate release of the system debugger. Its location on the Microsoft Download site changes every now and then, but right now it is located at <a href="http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx</a> for 32 bit platforms or <a href="http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx</a> for 64 bit platforms.
  * Start Windbg and attach it to your application.
  * Load SOS: <tt>.loadby sos mscorks</tt>
  * Dump the heap stats:<tt>!dumpheap -stats</tt>

The heap stats will dump all the types along with the number of objects allocated and the the total memory consumed, sorted by memory consumed. Usually the last entry in the list is type that is being leaked. In my case some 2 million objects were present so the leak was quite obvious.

One you tracked down the type being leaked it is usually very easy to find the cause and fix it just by a simple code analysis, but if you are having problems with that the solution is to find a few objects of the leaked type (use <tt>!dumpheap -type <LeakedTypeName></tt>) and then dump the tree references that prevents the Garbage Collector from reclaiming the object: <tt>!gcroot <AddressOfObject></tt>. There is an excellent post on the subject from Rico Mariani that details the process: <a href="http://blogs.msdn.com/ricom/archive/2004/12/10/279612.aspx" target="_blank">http://blogs.msdn.com/ricom/archive/2004/12/10/279612.aspx</a>.