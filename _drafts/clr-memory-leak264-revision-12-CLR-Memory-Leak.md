---
id: 276
title: CLR Memory Leak
date: 2009-01-19T13:55:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/01/19/264-revision-12/
permalink: /2009/01/19/264-revision-12/
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

In my case the leak cause was very obvious and was caused by not removing an event handler from an object event after the object was no longer needed. Unfortunately this pattern of leak is very frequent because of the default event handling code generated by pressing the Tab key in Visual Studio.

The following snippet of code illustrates the problem I&#8217;m talking about:

<pre><span style="color: Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">class</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">Worker<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Work(</span><span style="color:Blue">object</span><span style="color:Black">&nbsp;state)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Green">//&nbsp;do&nbsp;some&nbsp;work&nbsp;here&nbsp;...<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OnFinished();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;OnFinished()<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;(</span><span style="color:Blue">null</span><span style="color:Black">&nbsp;!=&nbsp;Finished)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Finished(</span><span style="color:Blue">this</span><span style="color:Black">,&nbsp;</span><span style="color:ff2b91af">EventArgs</span><span style="color:Black">.Empty);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">event</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">EventHandler</span><span style="color:Black">&nbsp;Finished;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;RunningCount;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">AutoResetEvent</span><span style="color:Black">&nbsp;eventDone;<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Main(</span><span style="color:Blue">string</span><span style="color:Black">[]&nbsp;args)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RunningCount&nbsp;=&nbsp;10000;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">AutoResetEvent</span><span style="color:Black">(</span><span style="color:Blue">false</span><span style="color:Black">);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">for</span><span style="color:Black">&nbsp;(</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;i&nbsp;=&nbsp;0;&nbsp;i&nbsp;&lt;&nbsp;10000;&nbsp;++i)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:ff2b91af">Worker</span><span style="color:Black">&nbsp;leaked&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">Worker</span><span style="color:Black">();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;leaked.Finished&nbsp;+=&nbsp;</span><span style="background-color:Yellow"><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">EventHandler</span><span style="color:Black">(leaked_Finished);</span></span><br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span><span style="color:ff2b91af">ThreadPool</span><span style="color:Black">.QueueUserWorkItem(</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">WaitCallback</span><span style="color:Black">(leaked.Work));<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone.WaitOne();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;leaked_Finished(</span><span style="color:Blue">object</span><span style="color:Black">&nbsp;sender,&nbsp;</span><span style="color:ff2b91af">EventArgs</span><span style="color:Black">&nbsp;e)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;count&nbsp;=&nbsp;</span><span style="color:ff2b91af">Interlocked</span><span style="color:Black">.Decrement(</span><span style="color:Blue">ref</span><span style="color:Black">&nbsp;RunningCount);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;(0&nbsp;==&nbsp;count)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone.Set();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
</span>
</pre>

To correct the problem in my example you have to remove the handler from the Worker Finiched event when it completes:

<pre><span style="color: Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">class</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">Worker<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Work(</span><span style="color:Blue">object</span><span style="color:Black">&nbsp;state)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Green">//&nbsp;do&nbsp;some&nbsp;work&nbsp;here&nbsp;...<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;OnFinished();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;OnFinished()<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;(</span><span style="color:Blue">null</span><span style="color:Black">&nbsp;!=&nbsp;Finished)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Finished(</span><span style="color:Blue">this</span><span style="color:Black">,&nbsp;</span><span style="color:ff2b91af">EventArgs</span><span style="color:Black">.Empty);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">event</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">EventHandler</span><span style="color:Black">&nbsp;Finished;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;RunningCount;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">AutoResetEvent</span><span style="color:Black">&nbsp;eventDone;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">EventHandler</span><span style="color:Black">&nbsp;callbackFinished;<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Main(</span><span style="color:Blue">string</span><span style="color:Black">[]&nbsp;args)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;RunningCount&nbsp;=&nbsp;10000;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">AutoResetEvent</span><span style="color:Black">(</span><span style="color:Blue">false</span><span style="color:Black">);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;span stcallbackFinished&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">EventHandler</span><span style="color:Black">(leaked_Finished);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">for</span><span style="color:Black">&nbsp;(</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;i&nbsp;=&nbsp;0;&nbsp;i&nbsp;&lt;&nbsp;10000;&nbsp;++i)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:ff2b91af">Worker</span><span style="color:Black">&nbsp;leaked&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">Worker</span><span style="color:Black">();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;leaked.Finished&nbsp;+=&nbsp;callbackFinished;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:ff2b91af">ThreadPool</span><span style="color:Black">.QueueUserWorkItem(</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:ff2b91af">WaitCallback</span><span style="color:Black">(leaked.Work));<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone.WaitOne();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;leaked_Finished(</span><span style="color:Blue">object</span><span style="color:Black">&nbsp;sender,&nbsp;</span><span style="color:ff2b91af">EventArgs</span><span style="color:Black">&nbsp;e)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:ff2b91af">Worker</span><span style="color:Black">&nbsp;leaked&nbsp;=&nbsp;(</span><span style="color:ff2b91af">Worker</span><span style="color:Black">)sender;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;leaked.Finished&nbsp;-=&nbsp;callbackFinished;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;count&nbsp;=&nbsp;</span><span style="color:ff2b91af">Interlocked</span><span style="color:Black">.Decrement(</span><span style="color:Blue">ref</span><span style="color:Black">&nbsp;RunningCount);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;(0&nbsp;==&nbsp;count)<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;eventDone.Set();<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;}</span>
</pre>