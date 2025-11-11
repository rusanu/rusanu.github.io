---
id: 80
title: Chained Updates
date: 2008-04-09T21:59:54+00:00
author: remus
layout: post
guid: /2008/04/09/chained-updates/
permalink: /2008/04/09/chained-updates/
categories:
  - Samples
---
One of the interesting features of the OUTPUT clauses introduced in SQL Server 2005 is that one can actualy chain DML statements into one complex statement that operates updates on several tables at once. Say we have a table with customer data and a process that has to bill each customer periodically. The &#8216;billing&#8217; process consist of an update on the table (say extend the subscription date), but the billing has to be processed separately. Consider an example where the processing involves a Web call to a bank portal to charge a credit card and, like all HTTP calls, it has the potential to fail. So we have two tables like this:

<!--more-->

<pre><code class="prettyprint lang-sql">
-- The customer data. Each customer has to be billed
-- when the expiration_date is passed
--
create table [customer_data] (
      [customer_id] int identity (1,1) not null,
      [name] nchar(256) not null,
      [account_no] nchar(80) not null,
      [expiration_date] datetime not null,
      [status] nchar(25) not null,
      constraint [customer_data_pk] primary key ([customer_id]));
go

create index [customer_data_expiration_date] on [customer_data] ([expiration_date]);
go

-- The processing queue
--
create table [bills] (
      [bill_id] int identity(1,1) not null,
      [customer_id] int,
      [enqueue_date] datetime not null,
      [next_retry] datetime not null,
      constraint [bills_pk] primary key nonclustered ([bill_id]),
      constraint [bills_cdx] unique clustered ([next_retry], [bill_id]));
go
</code></pre>

The &#8216;billing&#8217; operation has to update the status and expiration\_date on customer\_data and insert the pending billing requests to the bills table. Using the OUTPUT clause of the UPDATE we can actually do both operations in one single step:

<pre><code class="prettyprint lang-sql">
declare @now datetime;
set @now = GETDATE();
with cte_billable_customers as (
      select top(100)
                  [customer_id],
                  [status],
                  [expiration_date]
            from [customer_data] with (UPDLOCK)
            where [expiration_date] &lt; @now
            order by [expiration_date])
update cte_billable_customers
      set [status] = 'Pending',
            [expiration_date] = DATEADD(month, 1, @now)
      output INSERTED.[customer_id],
            @now as [enqueue_date],
            @now as [next_retry]
      into [bills] (
            [customer_id],
            [enqueue_date],
            [next_retry]);
</code></pre>

The TOP(100) has the role of restricting the processing into small batches to prevent a runaway update that might block the entire table.

So what advantages does such a construct offer? Personaly I like the fact that is one single statement instead of a pair of statements that uses an intermediary table variable or temporary table. The gives better isolation semantics for the locks acquired, it guarantees an atomic operation even if not called in a transaction context and it generates one single execution tree for the entire operation (both the update and the insert). It also has a certain coolnes factor.

As it is right now this is not much more other than a trivia alternate on your T-SQL bag'o'tricks. This sort of construct would potentially be much more powerful if further updates could be chained. That is if the INTO clause could have its own OUTPUT to be chained into an update, a delete or another insert. Potentialy building up chains of tens of table operations in one single statement. The would allow for entire procedures that today are declared in procedural fashion (statement after statement in T-SQL batches) to be writen in a SQL declarative fashion, leaving way for the optimizer to find shortcuts impossible to find today with separate operations. Unfortunately the language does not permit such constructs.

And now a piece of trivia: can you name the difference between an UPDLOCK and an XLOCK? I recently realized that I actualy forgot the difference. the answer is that UPDLOCK is asymmetric, it is compatible with S locks but S lock is not compatible with UPDLOCK. This kind of asymmetry helps in resolving the most common form of deadlock: a reader reads a row and then updates the same row (the so called 'read-write deadlock'). Because of its asymmetry the 'read' part can read the value while other readers are holding an S lock, but it prevents further readers from getting that row. When the update is applied, the UPDLOCK is upgraded to an XLOCK.

And this finally explains why my example contains an UPDLOCK hint in the SELECT statement: to prevent deadlocks between two concurent statements trying to process the customer_data table.