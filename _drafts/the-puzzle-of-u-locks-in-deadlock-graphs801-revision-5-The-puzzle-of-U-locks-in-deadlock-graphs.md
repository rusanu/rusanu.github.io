---
id: 807
title: The puzzle of U locks in deadlock graphs
date: 2010-05-12T14:12:47+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/05/12/801-revision-5/
permalink: /2010/05/12/801-revision-5/
---
In a <a href="http://stackoverflow.com/questions/2814377/deadlock-problem-because-of-an-update-lock/2814743#2814743" target="_blank">stackoverflow.com question</a> the user has asked how come a SELECT statement could own a U mode lock?

<div id="attachment_802" style="width: 310px" class="wp-caption alignnone">
  <a href="http://test.rusanu.com/wp-content/uploads/2010/05/deadlock-sux.png"><img src="http://rusanu.com/wp-content/uploads/2010/05/deadlock-sux-300x62.png" alt="S-U-X deadlock graph" title="deadlock-sux" width="300" height="62" class="size-medium wp-image-802" /></a>
  
  <p class="wp-caption-text">
    S-U-X deadlock graph
  </p>
</div>

The deadlock indeed suggests that the deadlock victim, a SELECT statement, is owning an U lock on the PK_B index. Why would a SELECT own an U lock? The query had no table hints and was a standalone query, not part of a multi-statement transaction that could had aquired the U lock in previous staements.

Turns out that the SELECT was actually **not** owning any U lock. The deadlock graph files (the *.xdl files) are in fact XML files and they can be opened as XML and inspected, for a little more detail than the visual deadlock graph visualizer permits. Here is the actual resource list in the deadlock XML:

<pre class="csharpcode"><span class="kwrd">&lt;</span><span class="html">resource-list</span><span class="kwrd">&gt;</span>
   <span class="kwrd">&lt;</span><span class="html">keylock</span> <span class="attr">hobtid</span><span class="kwrd">="72057594052411392"</span> <span class="attr">dbid</span><span class="kwrd">="10"</span>
         <span class="attr">objectname</span><span class="kwrd">="A"</span> <span class="attr">indexname</span><span class="kwrd">="PK_A"</span> <span class="attr">id</span><span class="kwrd">="lock17ed4040"</span> <span class="attr">mode</span><span class="kwrd">="X"</span> <span class="attr">associatedObjectId</span><span class="kwrd">="72057594052411392"</span><span class="kwrd">&gt;</span>
    <span class="kwrd">&lt;</span><span class="html">owner-list</span><span class="kwrd">&gt;</span>
     <span class="kwrd">&lt;</span><span class="html">owner</span> <span class="attr">id</span><span class="kwrd">="process4f5d000"</span> <span class="attr">mode</span><span class="kwrd">="X"</span><span class="kwrd">/&gt;</span>
    <span class="kwrd">&lt;/</span><span class="html">owner-list</span><span class="kwrd">&gt;</span>
    <span class="kwrd">&lt;</span><span class="html">waiter-list</span><span class="kwrd">&gt;</span>
     <span class="kwrd">&lt;</span><span class="html">waiter</span> <span class="attr">id</span><span class="kwrd">="processfa3c8e0"</span> <span class="attr">mode</span><span class="kwrd">="S"</span> <span class="attr">requestType</span><span class="kwrd">="wait"</span><span class="kwrd">/&gt;</span>
    <span class="kwrd">&lt;/</span><span class="html">waiter-list</span><span class="kwrd">&gt;</span>
   <span class="kwrd">&lt;/</span><span class="html">keylock</span><span class="kwrd">&gt;</span>
   <span class="kwrd">&lt;</span><span class="html">keylock</span> <span class="attr">hobtid</span><span class="kwrd">="72057594051166208"</span> <span class="attr">dbid</span><span class="kwrd">="10"</span>
        <span class="attr">objectname</span><span class="kwrd">="B"</span> <span class="attr">indexname</span><span class="kwrd">="PK_B"</span> <span class="attr">id</span><span class="kwrd">="lock22ea3940"</span>
        <span class="attr">mode</span><span class="kwrd">="U"</span> <span class="attr">associatedObjectId</span><span class="kwrd">="72057594051166208"</span><span class="kwrd">&gt;</span>
    <span class="kwrd">&lt;</span><span class="html">owner-list</span><span class="kwrd">&gt;</span>
     <span class="kwrd">&lt;</span><span class="html">owner</span> <span class="attr">id</span><span class="kwrd">="processfa3c8e0"</span> <span class="attr">mode</span><span class="kwrd">="S"</span><span class="kwrd">/&gt;</span>
    <span class="kwrd">&lt;/</span><span class="html">owner-list</span><span class="kwrd">&gt;</span>
    <span class="kwrd">&lt;</span><span class="html">waiter-list</span><span class="kwrd">&gt;</span>
     <span class="kwrd">&lt;</span><span class="html">waiter</span> <span class="attr">id</span><span class="kwrd">="process4f5d000"</span> <span class="attr">mode</span><span class="kwrd">="X"</span> <span class="attr">requestType</span><span class="kwrd">="convert"</span><span class="kwrd">/&gt;</span>
    <span class="kwrd">&lt;/</span><span class="html">waiter-list</span><span class="kwrd">&gt;</span>
   <span class="kwrd">&lt;/</span><span class="html">keylock</span><span class="kwrd">&gt;</span>
  <span class="kwrd">&lt;/</span><span class="html">resource-list</span><span class="kwrd">&gt;</span></pre>

As you can see, the resource lock22ea3940 is owned by the process processfa3c8e0 (the SELECT) indeed, but is owned in **S** mode. The process process4f5d000 (the UPDATE) is requesting this resource for a **convert** from U to X mode. So the true deadlock is like this:

  * SELECT owns a lock on the row in PK_B in S mode
  * SELECT wants a lock on the row in PK_A in S mode
  * UPDATE owns a lock on the row in PK_A in X mode
  * UPDATE also owns a U lock on the PK_B row. (S and U modes are compatible)
  * UPDATE is requesting a convert of the U lock it has on the row on PK_B to X mode

As you can see, there is no mysterious U lock owned by the SELECT. There is an U lock on the row in PK_B, but is owned by the UPDATE, which is requesting a convert to X for it. The fact that the resource is showned in the deadlock graph viewer in SSMS as being &#8216;Owner mode: U&#8217; and pointing to the SELECT is simply an artifact of how SSMS displays the deadlock graph.

The lesson to take home is that the visual graphic deadlock graph display is usefull only to have a cursory glance at the deadlock cycle. The true meat and potatoes are in the XML, which has a lot more information. Not to mention that the information in the XML is actually correct, which helps investigation&#8230;