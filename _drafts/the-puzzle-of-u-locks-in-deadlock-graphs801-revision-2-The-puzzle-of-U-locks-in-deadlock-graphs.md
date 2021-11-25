---
id: 804
title: The puzzle of U locks in deadlock graphs
date: 2010-05-12T13:55:40+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/05/12/801-revision-2/
permalink: /2010/05/12/801-revision-2/
---
In a <a href="http://stackoverflow.com/questions/2814377/deadlock-problem-because-of-an-update-lock/2814743#2814743" target="_blank">stackoverflow.com question</a> the user has asked how come a SELECT statement could own a U mode lock?

<div id="attachment_802" style="width: 310px" class="wp-caption alignnone">
  <a href="http://test.rusanu.com/wp-content/uploads/2010/05/deadlock-sux.png"><img src="http://rusanu.com/wp-content/uploads/2010/05/deadlock-sux-300x62.png" alt="S-U-X deadlock graph" title="deadlock-sux" width="300" height="62" class="size-medium wp-image-802" /></a>
  
  <p class="wp-caption-text">
    S-U-X deadlock graph
  </p>
</div>

The deadlock indeed suggests that the deadlock victim, a SELECT statement, is owning an U lock on the PK_B index. Why would a SELECT own an U lock? The query had no table hints and was a standalone query, not part of a multi-statement transaction that could had aquired the U lock in previous staements.

Turns out that the SELECT was actually **not** owning any U lock. The deadlock graph files (the *.xdl files) are in fact XML files and they can be opened as XML and inspected, for a little more detail than the visual deadlock graph visualizer permits