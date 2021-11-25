---
id: 686
title: Using tables as Queues
date: 2010-03-06T12:23:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/06/682-revision-4/
permalink: /2010/03/06/682-revision-4/
---
A very common question asked on all programming forums is how to implement queues based on database tables. This is not a trivial question actually. Implementing a queue backed by a table is notoriously difficult, error prone and susceptible to deadlocks. Because queues are usually needed as a link between various processing stages in a workflow they operate in highly concurrent environments where multiple processes enqueue rows into the table while multiple processes attempt to dequeue these rows. This concurrency creates correctness, scalability and performance challenges.

But since SQL Server 2005 introduced the OUTPUT clause, using tables as queues is no longer a hard problem. This fact is called out in the <a href="http://msdn.microsoft.com/en-us/library/ms177564.aspx" target="_blank">OUTPUT Clause<a /> topic in BOL</p> 

<h2>
  Heap Queue
</h2>

<p>
  The simplest queue is a heap: producers can equeue into the heap and consumers can dequeue, but order of operations is not important. The consumers can dequeue <i>any</i> row, as long it is unlocked:
</p>