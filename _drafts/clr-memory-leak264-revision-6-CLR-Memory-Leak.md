---
id: 270
title: CLR Memory Leak
date: 2009-01-19T13:34:02+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/01/19/264-revision-6/
permalink: /2009/01/19/264-revision-6/
---
On a recent client engagement I had to investigate what appeared a to be a memory leak in a managed application. The program was running for about a week now and it appeared to slowly degrade in performance over time. Although it appeared healthy at only about 245 MB of memory used, I decided to investigate. The fastest way in my opinion to track down leaks in a running environment is to attach Windbg and use SOS:

  * Download the appropriate release of the system debugger. Its location on the Microsoft Download site changes every now and then, but right now it is located at: 
      * <a href="http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx</a> for 32 bit platforms
      * <a href="http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx</a> for 64 bit platforms.
  * Start Windbg and attach it to your application.
  * Load SOS: 
    <pre>.loadby sos mscorks</pre>

  * Dump the heap stats: 
    <pre>!dumpheap -stats</pre>

The heap stats will dump all the types along with the number of objects allocated and the the total memory consumed, sorted by memory consumed. Usually the last entry in the list is type that is being leaked. In my case some 2 million objects were present so the leak was quite obvious.

One you tracked down the type being leaked it is usually very easy to find the cause and fix it just by a simple code analysis, but if you are having problems with that the solution is to find a few objects of the leaked type. Use: 

<pre>!dumpheap -type &lt;LeakedTypeName&gt;</pre>

and then dump the tree references that prevents the Garbage Collector from reclaiming the object: 

<pre>!gcroot &lt;AddressOfObject&gt;</pre>

There is an excellent post on the subject from Rico Mariani that details the process: <a href="http://blogs.msdn.com/ricom/archive/2004/12/10/279612.aspx" target="_blank">http://blogs.msdn.com/ricom/archive/2004/12/10/279612.aspx</a>.

There are also various SOS cheat sheets out there, the one I found refreshingly simple and succinct is this one: <a href="http://geekswithblogs.net/.netonmymind/archive/2006/03/14/72262.aspx" target="_blank">http://geekswithblogs.net/.netonmymind/archive/2006/03/14/72262.aspx</a>

In my case the leak cause was very obvious and was caused by not removing an event handler from an object event after the object was no longer needed. Unfortunately this pattern of leak is very frequent and