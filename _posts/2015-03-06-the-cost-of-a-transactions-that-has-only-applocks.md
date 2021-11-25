---
id: 2420
title: The cost of a transactions that has only applocks
date: 2015-03-06T05:23:23+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=2420
permalink: /2015/03/06/the-cost-of-a-transactions-that-has-only-applocks/
categories:
  - Tutorials
---
I recently seen a discussion on the merits of using SQL Server as an application lock manager. Application would start a transaction and acquire app locks, using [<tt>sp_getapplock</tt>](https://msdn.microsoft.com/en-us/library/ms189823.aspx). No actual data would be queried or modified in this transaction. The question is whether using SQL Server as such, basiclaly as an application lock manager, can lead to problems in SQL Server.

<p class="callout float-right">
  <tt>sp_getapplock</tt> uses the same internals for acquiring and granting locks like SQL Server data locks
</p>

Locks are relatively cheap. The engine can acquire, grant and release millions of them per second. A lock cost a small amount of memory, but overall locking is such an important and critical part of SQL Server operations that everything about locks is highly optimized. <tt>sp_getapplock</tt> is basically just a shell around the lock manager features and as such it gets a &#8216;free ride&#8217;. Highly optimized, efficient, fast and low overhead locks. However, this is not the only cost associated with app locks. **Transaction** scoped app locks must also play by the rules of transaction rollback and recovery, and one such rule is that exclusive locks acquired by transactions must be logged, since they must be re-acquired during recovery. App locks are no different, and this is easy to test:

    
    create database test;
    go
    
    alter database test set recovery  full;
    go
    
    backup database test to disk = 'nul:';
    go
    
    use test;
    go
    
    begin transaction
    exec sp_getapplock 'foo', 'Exclusive', 'Transaction'
    select * from sys.fn_dblog(null, null);
    

    
    00000021:0000005b:0001	LOP_BEGIN_XACT   ... user_transaction
    00000021:0000005b:0002	LOP_LOCK_XACT    ... ACQUIRE_LOCK_X APPLICATION: 5:0:[fo]:(ea53e177)
    

Sure enough our transaction that acquired an exclusive app lock has generated log records. This means that if the application is using SQL Server as a lock manager and holding long lived locks (for example while making a HTTP request) is also pinning the log, preventing truncation. Another side effect is log traffic, if the application is requesting lots of app locks it may result in noticeable log write throughput.

<p class="callout float-right">
  <tt>tempdb</tt> has a simplified logging model
</p>

I mentioned that the requirement to log the exclusive locks acquired in a transaction is a needed for recovery. Since <tt>tempdb</tt> is never recovered, <tt>tempdb</tt> has a more lax logging model. Specifically, it is not required to log exclusive locks acquired by transactions. So if you repeat the same experiment as above, but in <tt>tempdb</tt> you&#8217;ll see that no log record is generated:

    
    use tempdb;
    go
    
    begin transaction
    exec sp_getapplock 'foo', 'Exclusive', 'Transaction'
    select * from sys.fn_dblog(null, null);
    

If you make sure the current context is set to <tt>tempdb</tt> when calling <tt>sp_getapplock</tt> then you get a high performance lock manager for your application, with little overhead for your SQL Server engine. This is actually trivial to achieve in practice:

    
    use test
    go
    
    begin transaction
    exec tempdb..sp_getapplock 'foo', 'Exclusive', 'Transaction'
    select * from sys.fn_dblog(null, null)
    

By simply using the 3 part name <tt>tempdb..sp_getapplock</tt> I have effectively suppressed the logging of the app lock acquired! Of course a word of caution: please make sure that you are OK with app locks not being logged. The important thing to consider is how will your application behave when doing an online recovery. If app locks were not logged your app may acquire a an app lock _while the data that was protected by that app lock is actually going through recovery_. In practice this is seldom the case to worry, but you have to consider and acknowledge the trade offs.