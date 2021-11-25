---
id: 1034
title: Read/Write deadlock
date: 2009-05-16T03:13:03+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/05/16/361-revision-2/
permalink: /2009/05/16/361-revision-2/
---
How does a simple SELECT deadlock with an UPDATE? Surprisingly, they can deadlock even on well tuned systems that does not do spurious table scans. The answer is very simple: when the read and the write use two distinct access paths to reach the same key and they use them in reverse order. Lets consider a simple example: we have a table with a clustered index and an non-clustered index. The reader (T1) seeks a key in the non-clustered index and then needs to look up the clustered index to retrieve an additional column required by the SELECT projection list. The writer (T2) is updating the clustered index and then needs to perform an index maintenance operation on the non-clustered index. So T1 holds an S lock for the key K on the non-clustered index and wants an S lock on the same key K on the clustered index. T2 has an X lock on the key K on the clustered index and wants an X lock on same key K on the non-clustered index. Deadlock, T1 will be chosen as a victim. So you see, there are no complex queries involved, no suboptimal scan operations, no lock escalation nor page locks involved. Simple, correctly written queries may deadlock if doing read/write operations on the same key on a table with two indexes. Lets show this in an example:

<pre><span style="color: Black"></span><span style="color:Blue">use</span><span style="color:Black">&nbsp;tempdb</span><span style="color:Gray">;<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">table</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">TestId&nbsp;</span><span style="color:Blue">int</span><span style="color:Black">&nbsp;</span><span style="color:Blue">identity</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">,</span><span style="color:Black">1</span><span style="color:Gray">),<br />
</span><span style="color:Black">		</span><span style="color:Blue">constraint</span><span style="color:Black">&nbsp;cdxTestDealockTestId&nbsp;</span><span style="color:Blue">primary</span><span style="color:Black">&nbsp;</span><span style="color:Blue">key</span><span style="color:Black">&nbsp;</span><span style="color:Blue">clustered</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">TestId</span><span style="color:Gray">),<br />
</span><span style="color:Black">	ColA&nbsp;</span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">10</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Gray">not</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null,<br />
</span><span style="color:Black">	ColB&nbsp;</span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">10</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Gray">not</span><span style="color:Black">&nbsp;</span><span style="color:Gray">null);<br />
</span><span style="color:Black">go<br />
<br />
</span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">index</span><span style="color:Black">&nbsp;idxTestDeadlockColA&nbsp;</span><span style="color:Blue">on</span><span style="color:Black">&nbsp;TestDeadlock</span><span style="color:Gray">(</span><span style="color:Black">ColA</span><span style="color:Gray">);<br />
</span><span style="color:Black">go</span>
</pre>

Now lets populate the table with some random data:

<pre><span style="color: Black"></span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
</span><span style="color:Blue">insert</span><span style="color:Black">&nbsp;</span><span style="color:Blue">into</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">ColA</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ColB</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">values</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Red">'A'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Red">'B'</span><span style="color:Gray">);<br />
<br />
</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@i&nbsp;</span><span style="color:Blue">int</span><span style="color:Gray">;<br />
</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@i&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0</span><span style="color:Gray">;<br />
</span><span style="color:Blue">while</span><span style="color:Black">&nbsp;@i&nbsp;</span><span style="color:Gray">&lt;</span><span style="color:Black">&nbsp;10000<br />
</span><span style="color:Blue">begin<br />
</span><span style="color:Black">	</span><span style="color:Blue">insert</span><span style="color:Black">&nbsp;</span><span style="color:Blue">into</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">ColA</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ColB</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">values</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Fuchsia">rand</span><span style="color:Gray">()*</span><span style="color:Black">99999</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">rand</span><span style="color:Gray">()*</span><span style="color:Black">99999</span><span style="color:Gray">);<br />
</span><span style="color:Black">	</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@i&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@i</span><span style="color:Gray">+</span><span style="color:Black">1</span><span style="color:Gray">;<br />
</span><span style="color:Blue">end<br />
</span><span style="color:Black">go</span>
</pre>

Now open two query windows, and on the first one start this loop (the Reader):

<pre><span style="color: Black"></span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
</span><span style="color:Blue">while</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Blue">begin<br />
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@B&nbsp;</span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">10</span><span style="color:Gray">);<br />
</span><span style="color:Black">	</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@B&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;ColB&nbsp;</span><span style="color:Blue">from</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;ColA</span><span style="color:Gray">=</span><span style="color:Red">'A'</span><span style="color:Gray">;<br />
</span><span style="color:Blue">end</span>
</pre>

On the second one, start this loop (the Writer):

<pre><span style="color: Black"></span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
</span><span style="color:Blue">while</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Blue">begin<br />
</span><span style="color:Black">	</span><span style="color:Blue">update</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;ColA</span><span style="color:Gray">=</span><span style="color:Red">'A'</span><span style="color:Black">&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;TestId</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">update</span><span style="color:Black">&nbsp;TestDeadlock&nbsp;</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;ColA</span><span style="color:Gray">=</span><span style="color:Red">'B'</span><span style="color:Black">&nbsp;</span><span style="color:Blue">where</span><span style="color:Black">&nbsp;TestId</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">;<br />
</span><span style="color:Blue">end</span>
</pre>

Now switch back to the first query window and there you have it:

<pre><span style="color:Red">Msg 1205, Level 13, State 51, Line 6
Transaction (Process ID 52) was deadlocked on lock resources with another process
and has been chosen as the deadlock victim. Rerun the transaction.</span>
</pre>

Looking at the deadlock graph will show exactly the situation I described:

[<img src="http://rusanu.com/wp-content/uploads/2009/05/testdeadlock-300x61.png" alt="" title="testdeadlock" width="300" height="61" class="alignnone size-medium wp-image-362" />](http://test.rusanu.com/wp-content/uploads/2009/05/testdeadlock.png)

SPID 52 has an KEY S lock on idxTestDeadlockColA and wants an KEY S lock on cdxTestDeadlockTestId, SPID 53 has the KEY X lock on cdxTestDeadlockTestId and wants a KEY X lock on dxTestDeadlockColA. SPID 52 is chosen as victim since it performed less writes than SPID 53.

Why is this kind of deadlock not more prevalent? The key ingredient is that both T1 and T2 have to go after the same &#8216;key&#8217;. On a real life situation, this is just a matter of probabilities. If the write application pattern is to write new keys and to look up old ones, it is unlikely to happen. If the number of keys is very large then the probability of overlap is low (of course it increases around &#8216;hot spots&#8217;, for example a hot post on your website). In my recent experience on of the main culprits for these deadlocks are the hit counters prevalent in many content management systems (increment number of hits a page or image was shown).

Unfortunately things are not always so easy to understand and to explain. The same problem can manifest itself in the disguise of a join. The key learning to take home is this: neither the reads nor the writes do not acquire locks in an atomic fashion on all indexes of a table (that would be impossible, of course!). Whenever two transactions acquire these locks in a different order from one another, they open themselves to a deadlock.

The proper fix varies, of course. Things to consider are:

  * eliminate unnecessary columns from reader&#8217;s projection so he does not have to look up the clustered index
  * add required columns as contained columns to the non-clustered index to make the index covering, again so that the reader does not have look up the clustered index
  * avoid updates that have to maintain the non-clustered index
One final note: why did I add 10000 dummy record in my example? Because otherwise the optimizer would see that the clustered index has only one page and would choose a plan that scans this index instead of seeking the non-clustered index.