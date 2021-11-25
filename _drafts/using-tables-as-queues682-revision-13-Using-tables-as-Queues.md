---
id: 695
title: Using tables as Queues
date: 2010-03-06T18:35:21+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/06/682-revision-13/
permalink: /2010/03/06/682-revision-13/
---
A very common question asked on all programming forums is how to implement queues based on database tables. This is not a trivial question actually. Implementing a queue backed by a table is notoriously difficult, error prone and susceptible to deadlocks. Because queues are usually needed as a link between various processing stages in a workflow they operate in highly concurrent environments where multiple processes enqueue rows into the table while multiple processes attempt to dequeue these rows. This concurrency creates correctness, scalability and performance challenges.

But since SQL Server 2005 introduced the OUTPUT clause, using tables as queues is no longer a hard problem. This fact is called out in the <a href="http://msdn.microsoft.com/en-us/library/ms177564.aspx" target="_blank">OUTPUT Clause</a> topic in BOL:

> You can use OUTPUT in applications that use tables as queues, or to hold intermediate result sets. That is, the application is constantly adding or removing rows from the table&#8230; Other semantics may also be implemented, such as using a table to implement a stack.

The reason why OUTPUT clause is critical is that it offers an atomic destructive read operation that allows us to remove the dequeued row _and_ return it to the caller, in one single statement.

## Heap Queue

The simplest queue is a heap: producers can equeue into the heap and consumers can dequeue, but order of operations is not important. Order is not important, the consumers can dequeue _any_ row, as long as it is unlocked:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">table&lt;/span>&lt;span style="color:Black">&nbsp;HeapQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;Payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_enqueueHeap&lt;br />
&nbsp;&nbsp;@payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;HeapQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">Payload&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@Payload&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_dequeueHeap&nbsp;&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">delete&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;HeapQueue&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">rowlock&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">readpast&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">output&lt;/span>&lt;span style="color:Black">&nbsp;deleted&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">payload&lt;/span>&lt;span style="color:Gray">;&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;br />
go&lt;/span>
&lt;/pre>
&lt;p></code>

A heap queue can satisfy most producer-consumer patterns. It scales well and is very simple to implement and understand. Notice the (rowlock, readpast) hints on the delete operation, they allow for concurrent consumers to dequeue rows from the table without blocking each other. **A heap queue makes no order guarantees**.

## FIFO Queue

When he queueing and dequeuing operations have to support a certain order two changes have to be made:

  * The table must be organized as a clustered index ordered by a key that preserves the desired dequeue order.
  * The dequeue operation must contain an ORDER BY clause to guarantee the order.