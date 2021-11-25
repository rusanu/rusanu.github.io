---
id: 735
title: Using tables as Queues
date: 2010-03-24T23:42:51+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/24/682-revision-31/
permalink: /2010/03/24/682-revision-31/
---
A very common question asked on all programming forums is how to implement queues based on database tables. This is not a trivial question actually. Implementing a queue backed by a table is notoriously difficult, error prone and susceptible to deadlocks. Because queues are usually needed as a link between various processing stages in a workflow they operate in highly concurrent environments where multiple processes enqueue rows into the table while multiple processes attempt to dequeue these rows. This concurrency creates correctness, scalability and performance challenges.

But since SQL Server 2005 introduced the OUTPUT clause, using tables as queues is no longer a hard problem. This fact is called out in the <a href="http://msdn.microsoft.com/en-us/library/ms177564.aspx" target="_blank">OUTPUT Clause</a> topic in BOL:

> You can use OUTPUT in applications that use tables as queues, or to hold intermediate result sets. That is, the application is constantly adding or removing rows from the table&#8230; Other semantics may also be implemented, such as using a table to implement a stack.

The reason why OUTPUT clause is critical is that it offers an atomic destructive read operation that allows us to remove the dequeued row _and_ return it to the caller, in one single statement. In this article I want to 

  * [Heap Queues](#HeapQueues)
  * [Fifo Queues](#FifoQueues)
  * [Concurrency](#Concurrency)
  * [Stack Queues](#StackQueues)
  * [Pending Queues](#PendingQueues)

## <a name="HeapQueue">Heap Queues</a>

The simplest queue is a heap: producers can equeue into the heap and consumers can dequeue, but order of operations is not important: the consumers can dequeue _any_ row, as long as it is unlocked.

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

## <a name="FifoQueues">FIFO Queues</a>

When he queueing and dequeuing operations have to support a certain order two changes have to be made:

  * The table must be organized as a clustered index ordered by a key that preserves the desired dequeue order.
  * The dequeue operation must contain an ORDER BY clause to guarantee the order.

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">table&lt;/span>&lt;span style="color:Black">&nbsp;FifoQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;Id&nbsp;&lt;/span>&lt;span style="color:Blue">bigint&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">not&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">null&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">identity&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">),&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;Payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">clustered&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">index&lt;/span>&lt;span style="color:Black">&nbsp;cdxFifoQueue&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Black">&nbsp;FifoQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">Id&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_enqueueFifo&lt;br />
&nbsp;&nbsp;@payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;FifoQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">Payload&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@Payload&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_dequeueFifo&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;cte&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;Payload&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;FifoQueue&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">rowlock&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">readpast&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">order&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">by&lt;/span>&lt;span style="color:Black">&nbsp;Id&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">delete&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;cte&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">output&lt;/span>&lt;span style="color:Black">&nbsp;deleted&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">Payload&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;/span>
&lt;/pre>
&lt;p></code>

By adding the IDENTITY column to our queue and making it the clustered index, we can dequeue in the order inserted. The enqueue operation is identical with our Heap Queue, but the dequeue is slightly changed, as the requirement to dequeue in the order inserted means that we have to specify an ORDER BY. Since the DELETE statement does not support ORDER BY, we use a Common Table Expression to select the row to be dequeued, then delete this row and return the payload in the OUTPUT clause. Isn&#8217;t this the same as doing a SELECT followed by a DELETE, and hence exposed to the traditional correctness problems with table backed queues? Technically, it is. But this is a SELECT followed by a DELETE that actually _works_ for table based queues. Let me explain.

Because the query is actually an DELETE of a CTE, the query execution will occur as a DELETE, not as an SELECT followed by a DELETE, and also not as a SELECT executed in the context of the DELETE. The crucial part is that the SELECT part will aquire **LCK\_M\_U** update locks on the rows scanned. LCK\_M\_U is compatible with LCK\_M\_S shared locks, but is incompatible with another LCK\_M\_U. So two concurrent dequeue threads will **not** try to dequeue the same row. One will grab the first row free, the other thread will grab the next row.

Is also worth looking at how a compact plan the usp_dequeueFifo has:

<div id="attachment_699" style="width: 310px" class="wp-caption alignnone">
  <a href="http://test.rusanu.com/wp-content/uploads/2010/03/fifoqueueplan.png"><img src="http://rusanu.com/wp-content/uploads/2010/03/fifoqueueplan-300x60.png" alt="usp_dequeueFifo execution plan" title="fifoqueueplan" width="300" height="60" class="size-medium wp-image-699" /></a>
  
  <p class="wp-caption-text">
    usp_dequeueFifo execution plan
  </p>
</div>

Compare this with the alternative of using a subquery to locate the row to be deleted:

<div id="attachment_701" style="width: 310px" class="wp-caption alignnone">
  <a href="http://test.rusanu.com/wp-content/uploads/2010/03/subqueryplan.png"><img src="http://rusanu.com/wp-content/uploads/2010/03/subqueryplan-300x71.png" alt="Subquery deque plan" title="subqueryplan" width="300" height="71" class="size-medium wp-image-701" /></a>
  
  <p class="wp-caption-text">
    Subquery deque plan
  </p>
</div>

  

<code class="prettyprint lang-sql">
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">delete top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">FifoQueue&lt;br />
&lt;/span>&lt;span style="color:Blue">output &lt;/span>&lt;span style="color:Black">deleted&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">Payload&lt;br />
&lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">Id &lt;/span>&lt;span style="color:Gray">= (&lt;br />
&lt;/span>&lt;span style="color:Blue">select top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Black">Id&lt;br />
  &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">FifoQueue &lt;/span>&lt;span style="color:Blue">with &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">rowlock&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">updlock&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">readpast&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">order by &lt;/span>&lt;span style="color:Black">Id&lt;/span>&lt;span style="color:Gray">)&lt;/span>
&lt;/pre>
&lt;p></code>

### <a name="Concurrency">Strict Ordering and Concurrency</a>

Strict FIFO ordering in a database world would have to take into account transactions. If transaction T1 dequeues row A, transaction T2 dequeues the next row B and then T1 rolls back and T2 commits, the row B was processed out of order. So any dequeue operation would have to wait for the previous dequeue to **committ** before proceeding. While this is correct, is also highly inefficient, as it means that all transactions must serialize access to the queue. Many applications accept the processing to occur out of order for the sake of achieving a reasonable scalability and performance.

If strict FIFO order is _required_ then you have to **remove the readpast** hint from the usp_dequeueFifo procedure. When this is done, only one transaction can dequeue rows from the queue at a time. All other transaction will have to wait until the first one commits. This is not an implementation artifact, it is a fundamental requirement derived from the ACID properties of transactions.

If a lax FIFO order is acceptable, then the readpast hint will ensure that multiple transactions can dequeue rows concurrently. However, the strict FIFO order cannot be guaranteed in this case.

## <a name="LifoStacks">LIFO Stacks</a></h3> 

A stack backed by a queue is also possible. The implementation and table structure is almost identical with the FIFO queue, with one difference: the ORDER BY clause has a DESC.


<code class="prettyprint lang-sql">
&lt;pre>
&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;cte&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;Payload&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;FifoQueue&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">rowlock&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">readpast&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">order&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">by&lt;/span>&lt;span style="color:Black">&nbsp;Id&lt;/span>&lt;span style="color:Blue">&nbsp;DESC&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">delete&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;cte&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">output&lt;/span>&lt;span style="color:Black">&nbsp;deleted&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">Payload&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>

Because all operations (enqueue and dequeue) occur on the same rows, stacks implemented as tables tend to create a very hot spot on the page which currently contains these rows. Because all row insert, delete and update operations need to lock the page latch exclusively and stacks operate on the rows grouped at one end of the table, the result is high page latch contention on this page. Queues have the same problem, but to a lesser extent as the operations are spread in inserts at one end of the table and deletes at the other end, so the same number of operations is split into two hot spots instead of a single one like in stacks case.

## <a name="PendingQueues">Pending Queues</a>

Another category of queues are pending queues. Items are inserted with a due date, and the dequeue operation returns rows that are due at dequeue time. This type of queues is common in scheduling systems.


<code class="prettyprint lang-sql">
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">table&lt;/span>&lt;span style="color:Black">&nbsp;PendingQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;DueTime&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">not&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">null,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;Payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">clustered&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">index&lt;/span>&lt;span style="color:Black">&nbsp;cdxPendingQueue&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Black">&nbsp;PendingQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">DueTime&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_enqueuePending&lt;br />
&nbsp;&nbsp;@dueTime&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Gray">,&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;@payload&nbsp;&lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">insert&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">into&lt;/span>&lt;span style="color:Black">&nbsp;PendingQueue&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">DueTime&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;Payload&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">values&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@dueTime&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;@payload&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">procedure&lt;/span>&lt;span style="color:Black">&nbsp;usp_dequeuePending&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">nocount&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">declare&lt;/span>&lt;span style="color:Black">&nbsp;@now&nbsp;&lt;/span>&lt;span style="color:Blue">datetime&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">set&lt;/span>&lt;span style="color:Black">&nbsp;@now&nbsp;&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">();&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;cte&nbsp;&lt;/span>&lt;span style="color:Blue">as&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">select&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;/span>&lt;span style="color:Black">&nbsp;&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;Payload&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;PendingQueue&nbsp;&lt;/span>&lt;span style="color:Blue">with&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">rowlock&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">readpast&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">where&lt;/span>&lt;span style="color:Black">&nbsp;DueTime&nbsp;&lt;/span>&lt;span style="color:Gray">&lt;&lt;/span>&lt;span style="color:Black">&nbsp;@now&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">order&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">by&lt;/span>&lt;span style="color:Black">&nbsp;DueTime&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Black">&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">delete&lt;/span>&lt;span style="color:Black">&nbsp;&lt;/span>&lt;span style="color:Blue">from&lt;/span>&lt;span style="color:Black">&nbsp;cte&lt;br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/span>&lt;span style="color:Blue">output&lt;/span>&lt;span style="color:Black">&nbsp;deleted&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">Payload&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>

I choose to use UTC times for my example, and I highly recommend you do the same for your applications. Not only this eliminates the problem of having to deal with timezones, but also your pending operations will behave correctly on the two times each year when summer time enters into effect or when it ends.