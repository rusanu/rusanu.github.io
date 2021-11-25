---
id: 1116
title: Articles
date: 2016-01-07T12:38:54+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/05/20/896-autosave/
permalink: /2016/01/07/896-autosave-v1/
---
<div class="entry">
  <h3>
    Recommended Articles
  </h3>
  
  <dl>
    <dt>
      <a href="http://rusanu.com/2013/08/01/understanding-how-sql-server-executes-a-query/">Understanding how SQL Server executes a query</a>
    </dt>
    
    <dd>
      An explanation of how SQL Server executes a query. Read this if you are curious about the how SQL Server executes the T-SQL requests from your app, how queries and transactions work, how SQL Server reads and writes data, the query operator tree execution model.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2014/02/24/how-to-analyse-sql-server-performance/">How to analyse SQL Server performance</a>
    </dt>
    
    <dd>
      I tried to distill my performance troubleshooting know-how into this article. It covers many areas and goes quite deep into some of them. It presents specific techniques you can use to identify bottlenecks.
    </dd>
    
    <dt>
      Storing Images and Media on SQL Server with ASP.Net MVC
    </dt>
    
    <dd>
      A couple of articles that show how to efficiently stream large files stored in the database, like image files, avoiding large memory allocations in the ASP.NEt process by correctly using streaming semantics. The first article <a href="http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/">Download and Upload images from SQL Server via ASP.Net MVC</a> shows how to use a VARBINARY type column. The follow up article <a href="http://rusanu.com/2011/02/06/filestream-mvc-download-and-upload-images-from-sql-server/">FILESTREAM MVC: Download and Upload images from SQL Server</a> shows how to use a FILESTREAM column.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2009/08/05/asynchronous-procedure-execution/">Asynchronous procedure execution</a>
    </dt>
    
    <dd>
      How to reliably invoke procedures asynchronously in your database by means of Service Broker Activation. By leveraging the internal activation mechanism, your client does not have keep a connection open, nor does it have to rely on some sort of helping service process.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2005/12/20/troubleshooting-dialogs/">Troubleshooting Conversations</a>
    </dt>
    
    <dd>
      The original article published in 2005 that started this blog. Step by step advice to troubleshoot Service Broker message delivery. Still valid today.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2007/04/25/reusing-conversations/">Reusing Conversations</a>
    </dt>
    
    <dd>
      Getting the high message throughput in Service Broker depends on proper conversation management. This article shows a technique of reusing conversations based on session ID. Using this technique you can easily achieve throughputs of thousands of messages per send end-to-end between two sites.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2009/05/16/readwrite-deadlock/">Read-Write deadlock</a>
    </dt>
    
    <dd>
      A presentation of a common deadlock scenario between a simple SELECT and an UPDATE. This is common deadlock that happens in content management systems when updating the view count for images and posts.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2006/03/01/signing-an-activated-procedure/">Signing activated procedures</a>
    </dt>
    
    <dd>
      Because of the EXECUTE AS impersonation context activated procedures by default cannot access objects in other databases, like <tt>msdb.dbo.sp_send_mail</tt>, nor server scoped objects like linked servers. This article shows how to use code signing to overcome these problems.
    </dd>
    
    <dt>
      <a href="http://rusanu.com/2006/06/17/the-mysterious-notification/">The Mysterious Notification</a>
    </dt>
    
    <dd>
      Are you&#8217;re using SqlDependency on your site? If you want to learn more about how this feature works and the underlaying technology this article is good starter. A brief presentation into Query Notifications, SqlNotificationRequest and SqlDependency.
    </dd>
  </dl>
  
  <dt>
    <a href="http://rusanu.com/2010/08/04/sqldependency-based-caching-of-linq-queries/">SqlDependency based caching of LINQ Queries</a>
  </dt>
  
  <dd>
    <code class="prettyprint lang-sql">LinqToCache</code> is a library that adds SqlDependency based caching to LINQ queries.
  </dd>
  
  <dt>
    <a href="http://rusanu.com/2006/10/16/writing-service-broker-procedures/">Writing Service Broker Procedures</a>
  </dt>
  
  <dd>
    Originally published in 2006 this article goes over every technique of writing Service Broker activated procedures: single message, cursor, set oriented etc. It shows the basic skeleton of each one accompanied with performance measurements. If you are now only able to dequeue about 50 messages per second, read this article to understand how to get to 5000 messages per second.
  </dd>
</div>

### All Articles

<div class="Articles" style="font-size:80%">
  <ul class="all-posts">
    <li>
      <a href="http://rusanu.com/2017/11/24/sql_injection_cast_can_introduce_single_quotes/">SQL Injection: casting can introduce additional single quotes</a> <span class="post-date">2017 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2017/09/22/identifying-sqlconnection-objects-in-a-dump/">Identifying SqlConnection objects in a dump</a> <span class="post-date">2017 Sep</span>
    </li>
    <li>
      <a href="http://rusanu.com/2016/04/01/understanding-sql-server-query-store/">Understanding SQL Server Query Store</a> <span class="post-date">2016 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2016/03/18/introducing-dbhistory-com/">Introducing DBHistory.com</a> <span class="post-date">2016 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2015/03/06/the-cost-of-a-transactions-that-has-only-applocks/">The cost of a transactions that has only applocks</a> <span class="post-date">2015 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2014/06/17/sql-server-2014-updateable-columnstores-q-and-a/">SQL Server 2014 updateable columnstores Q and A</a> <span class="post-date">2014 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2014/05/30/windowsxray-is-made-public-as-media-experience-analyzer/">WindowsXRay is made public as Media eXperience Analyzer</a> <span class="post-date">2014 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2014/03/31/how-to-prevent-conversation-endpoint-leaks/">How to prevent conversation endpoint leaks</a> <span class="post-date">2014 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2014/03/10/how-to-read-and-interpret-the-sql-server-log/">How to read and interpret the SQL Server log</a> <span class="post-date">2014 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2014/02/24/how-to-analyse-sql-server-performance/">How to analyse SQL Server performance</a> <span class="post-date">2014 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/12/02/sql-server-clustered-columnstore-tuple-mover/">SQL Server clustered columnstore Tuple Mover</a> <span class="post-date">2013 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/11/19/using-hive-local-mode-with-yarn/">Using Hive local mode with YARN</a> <span class="post-date">2013 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/08/27/services-running-under-domain-account-should-add-dnscache-service-as-dependency/">Services running under domain account should add dnscache service as dependency</a> <span class="post-date">2013 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/08/01/understanding-how-sql-server-executes-a-query/">Understanding how SQL Server executes a query</a> <span class="post-date">2013 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/06/11/sql-server-clustered-columnstore-indexes-at-teched-2013/">SQL Server Clustered Columnstore Indexes at TechEd 2013</a> <span class="post-date">2013 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/02/15/registry-bloat-after-sql-server-2012-sp1-installation/">Registry bloat after SQL Server 2012 SP1 installation</a> <span class="post-date">2013 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/02/11/how-to-enable-selective-xml-indexes-in-sql-server-2012-sp1/">How to enable Selective XML indexes in SQL Server 2012 SP1</a> <span class="post-date">2013 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/02/06/how-to-enable-and-disable-a-queue-using-smo/">How to enable and disable a queue using SMO</a> <span class="post-date">2013 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2013/01/25/sql-server-backup-to-url/">SQL Server backup to URL</a> <span class="post-date">2013 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/11/23/case-sensitive-collation-sort-order/">Case Sensitive collation sort order</a> <span class="post-date">2012 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/10/15/handling-exceptions-that-occur-during-the-receive-statement-in-activated-procedures/">Handling exceptions that occur during the RECEIVE statement in activated procedures</a> <span class="post-date">2012 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/07/27/how-to-shrink-the-sql-server-log/">How to shrink the SQL Server log</a> <span class="post-date">2012 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/05/29/inside-the-sql-server-2012-columnstore-indexes/">Inside the SQL Server 2012 Columnstore Indexes</a> <span class="post-date">2012 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/03/17/1000-consecutive-days-on-stackoverflow/">1000 Consecutive days on StackOverflow</a> <span class="post-date">2012 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/02/16/adding-a-nullable-column-can-update-the-entire-table/">Adding a nullable column can update the entire table</a> <span class="post-date">2012 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/01/27/show-all-index-and-heap-access-operators-in-the-plan-cache/">Show all index and heap access operators in the plan cache</a> <span class="post-date">2012 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2012/01/17/what-is-an-lsn-log-sequence-number/">What is an LSN: Log Sequence Number</a> <span class="post-date">2012 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/10/20/sql-server-table-columns-under-the-hood/">SQL Server table columns under the hood</a> <span class="post-date">2011 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/10/19/understanding-hash-sort-and-exchange-spill-events/">Understanding Hash, Sort and Exchange Spill events</a> <span class="post-date">2011 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/08/10/t-sql-functions-do-no-imply-a-certain-order-of-execution/">T-SQL functions do no imply a certain order of execution</a> <span class="post-date">2011 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/08/05/online-index-operations-for-indexes-containing-lob-columns/">Online Index Operations for indexes containing LOB columns</a> <span class="post-date">2011 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/07/20/how-to-multicast-messages-with-sql-server-service-broker/">How to Multicast messages with SQL Server Service Broker</a> <span class="post-date">2011 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/07/13/online-non-null-with-values-column-add-in-sql-server-11/">Online non-NULL with values column add in SQL Server 2012</a> <span class="post-date">2011 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/07/13/how-to-update-a-table-with-a-columnstore-index/">How to update a table with a columnstore index</a> <span class="post-date">2011 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/07/13/how-to-use-columnstore-indexes-in-sql-server/">How to use columnstore indexes in SQL Server</a> <span class="post-date">2011 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/06/23/sigmod-2011-webcasts/">SIGMOD 2011 Webcasts</a> <span class="post-date">2011 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/06/01/scale-out-sql-server-by-using-reliable-messaging/">Scale out SQL Server by using Reliable Messaging</a> <span class="post-date">2011 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/04/04/how-to-determine-the-database-version-of-an-mdf-file/">How to determine the database version of an MDF file</a> <span class="post-date">2011 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/02/21/erlands-sommarskog-adds-another-gem-to-his-treasure-chest/">Erland's Sommarskog adds another gem to his treasure chest</a> <span class="post-date">2011 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/02/06/filestream-mvc-download-and-upload-images-from-sql-server/">FILESTREAM MVC: Download and Upload images from SQL Server</a> <span class="post-date">2011 Feb</span>
    </li>
    <li>
      <a href="http://rusanu.com/2011/01/15/how-to-pass-a-null-value-in-a-message-to-a-queue-in-sql-server/">How to pass a NULL value in a message to a queue in SQL Server</a> <span class="post-date">2011 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/12/28/download-and-upload-images-from-sql-server-with-asp-net-mvc/">Download and Upload images from SQL Server via ASP.Net MVC</a> <span class="post-date">2010 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/11/23/this-server-supports-version-662-and-earlier/">This server supports version 662 and earlier...</a> <span class="post-date">2010 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/11/22/try-catch-throw-exception-handling-in-t-sql/">TRY CATCH THROW: Error handling changes in T-SQL </a> <span class="post-date">2010 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/11/11/alwayson-high-availability-and-reads-scale-out/">AlwaysOn: High-Availability and reads Scale-Out</a> <span class="post-date">2010 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/08/04/sqldependency-based-caching-of-linq-queries/">SqlDependency based caching of LINQ Queries</a> <span class="post-date">2010 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/06/29/remote-desktop-manager-now-available/">Remote Desktop Manager now available</a> <span class="post-date">2010 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/06/11/high-volume-contiguos-real-time-audit/">High Volume Contiguos Real Time Audit and ETL</a> <span class="post-date">2010 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/05/12/the-puzzle-of-u-locks-in-deadlock-graphs/">The puzzle of U locks in deadlock graphs</a> <span class="post-date">2010 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/05/11/effective-speakers-at-portland-devsat-and-sqlsat27/">Effective Speakers at Portland #devsat and #sqlsat27</a> <span class="post-date">2010 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/04/28/sql-server-2008-r2-express-database-size-limit-10gb/">SQL Server 2008 R2 Express database size limit: 10GB</a> <span class="post-date">2010 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/04/23/how-to-change-database-mirroring-encryption-with-minimal-downtime/">How to change database mirroring encryption with minimal downtime</a> <span class="post-date">2010 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/04/01/what-is-remus-up-to/">What is Remus up to?</a> <span class="post-date">2010 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/03/31/the-bizzaro-guide-to-sql-server-performance/">The Bizzaro Guide to SQL Server Performance</a> <span class="post-date">2010 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/03/26/using-tables-as-queues/">Using tables as Queues</a> <span class="post-date">2010 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/03/22/performance-comparison-of-varcharmax-vs-varcharn/">Performance comparison of varchar(max) vs. varchar(N)</a> <span class="post-date">2010 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/03/09/dealing-with-large-queues/">Dealing with Large Queues</a> <span class="post-date">2010 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2010/01/12/what-deprecated-features-am-i-using/">What deprecated features am I using?</a> <span class="post-date">2010 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/11/22/system-pagefile-size-on-machines-with-large-ram/">System pagefile size on machines with large RAM</a> <span class="post-date">2009 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/10/28/bugcollectcom-better-customer-support/">bugcollect.com: better customer support</a> <span class="post-date">2009 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/10/26/select-count/">select count(*);</a> <span class="post-date">2009 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/09/28/asynchronous-t-sql-at-sql-saturday-26/">Asynchronous T-SQL at SQL Saturday #26</a> <span class="post-date">2009 Sep</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/09/13/on-sql-server-boolean-operator-short-circuit/">On SQL Server boolean operator short-circuit</a> <span class="post-date">2009 Sep</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/08/18/passing-parameters-to-a-background-procedure/">Passing Parameters to a Background Procedure</a> <span class="post-date">2009 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/08/05/asynchronous-procedure-execution/">Asynchronous procedure execution</a> <span class="post-date">2009 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/07/26/myspace-uses-sql-server-service-broker-to-protect-integrity-of-1-petabyte-of-data/">MySpace Uses SQL Server Service Broker to Protect Integrity of 1 Petabyte of Data</a> <span class="post-date">2009 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/07/24/fix-slow-application-startup-due-to-code-sign-validation/">Fix slow application startup due to code sign validation</a> <span class="post-date">2009 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/07/08/inspiration-is-perishable/">Inspiration is perishable</a> <span class="post-date">2009 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/06/11/exception-handling-and-nested-transactions/">Exception handling and nested transactions</a> <span class="post-date">2009 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/05/29/lockres-collision-probability-magic-marker-16777215/">%%lockres%% collision probability magic marker: 16,777,215</a> <span class="post-date">2009 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/05/21/have-you-met-i%c2%bb%c2%bf-say-hello-to-my-bom/">Have you met ï»¿ ? Say hello to my BOM</a> <span class="post-date">2009 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/05/18/stackoverflowcom-how-to-execute-well-on-a-good-idea/">stackoverflow.com: how to execute well on a good idea</a> <span class="post-date">2009 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/05/16/readwrite-deadlock/">Read/Write deadlock</a> <span class="post-date">2009 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/05/15/version-control-and-your-database/">Version Control and your Database</a> <span class="post-date">2009 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/04/18/a-fix-for-error-cannot-find-the-remote-service-sqlquerynotificationservice-guid/">A fix for error Cannot find the remote service SqlQueryNotificationService-GUID</a> <span class="post-date">2009 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/04/11/using-xslt-to-generate-performance-counters-code/">Using XSLT to generate Performance Counters code</a> <span class="post-date">2009 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/03/25/service-broker-whitepaper-on-msdn-the-150-trick/">Service Broker Whitepaper on MSDN: the 150 trick</a> <span class="post-date">2009 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/03/24/databasejournal-tutorial/">DatabaseJournal Tutorial</a> <span class="post-date">2009 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/03/20/things-i-know-now-blogging-can-get-you-into-a-email-ponzi-scheme/">Things I know now: blogging can get you into a email ponzi scheme</a> <span class="post-date">2009 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2009/01/19/clr-memory-leak/">CLR Memory Leak</a> <span class="post-date">2009 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/12/16/bogus-error-833-issue-is-fixed-in-sp3/">Bogus error 833 issue is fixed in SP3</a> <span class="post-date">2008 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/11/26/replacing-service-certificates-that-are-near-expiration/">Replacing Service Certificates that are near expiration</a> <span class="post-date">2008 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/11/11/high-performance-windows-programs/">High Performance Windows programs</a> <span class="post-date">2008 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/11/05/reusing-conversations-a-better-mouse-trap/">Reusing Conversations: a Better Mouse Trap</a> <span class="post-date">2008 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/11/04/conversations-authentication/">Conversations Authentication</a> <span class="post-date">2008 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/10/28/event-id-833-io-requests-taking-longer-than-15-seconds/">Event ID 833: I/O requests taking longer than 15 seconds</a> <span class="post-date">2008 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/10/25/replacing-endpoint-certificates-that-are-near-expiration/">Replacing Endpoint Certificates that are near expiration</a> <span class="post-date">2008 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/10/23/how-does-certificate-based-authentication-work/">How does Certificate based Authentication work</a> <span class="post-date">2008 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/09/14/wcf-channel-for-ssb/">WCF Channel for SSB</a> <span class="post-date">2008 Sep</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/09/09/quest-service-broker-admin-feedback/">Quest Service Broker Admin feedback</a> <span class="post-date">2008 Sep</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/08/25/certificate-not-yet-valid/">Certificate Not Yet Valid</a> <span class="post-date">2008 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/08/13/error-handling-and-activation/">Error Handling and Activation</a> <span class="post-date">2008 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/08/08/service-broker-team-blog/">Service Broker Team Blog</a> <span class="post-date">2008 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/08/05/database-journal-articles-on-service-broker/">Database Journal articles on Service Broker</a> <span class="post-date">2008 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/08/03/understanding-queue-monitors/">Understanding Queue Monitors</a> <span class="post-date">2008 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/07/16/ot-terrarium-2-released-in-the-wild/">OT: Terrarium 2 released in the wild</a> <span class="post-date">2008 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/05/22/dialog-security-configuration-wizard-in-toad/">Dialog Security Configuration Wizard in Toad</a> <span class="post-date">2008 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/05/16/viable-tools-for-service-broker/">Viable tools for Service Broker</a> <span class="post-date">2008 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/04/15/sqlpass-2008-slides-and-demo/">SQLPASS 2008 Slides and Demo</a> <span class="post-date">2008 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/04/13/service-broker-administration-monitoring-and-troubleshooting/">Service Broker Administration, Monitoring and Troubleshooting</a> <span class="post-date">2008 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/04/09/chained-updates/">Chained Updates</a> <span class="post-date">2008 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/01/23/speaker-at-european-pass-conference-2008/">Speaker at European PASS Conference 2008</a> <span class="post-date">2008 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2008/01/04/sqldependencyonchange-callback-timing/">SqlDependency.OnChange callback timing</a> <span class="post-date">2008 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/12/11/pro-sql-server-2008-service-broker-2/">Pro SQL Server 2008 Service Broker</a> <span class="post-date">2007 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/12/03/resending-messages/">Resending messages</a> <span class="post-date">2007 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/12/03/data-mining-event/">Data Mining Event</a> <span class="post-date">2007 Dec</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/28/troubleshooting-dialogs-the-sequel/">Troubleshooting dialogs, the sequel</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/28/is-service-broker-in-sql-server-2008-compatible-with-the-one-in-sql-server-2005/">Is Service Broker in SQL Server 2008 compatible with the one in SQL Server 2005?</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/27/the-latest-sql-server-2008-ctp/">The latest SQL Server 2008 CTP</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/11/prevent-errorlog-growth/">Prevent ERRORLOG growth</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/10/when-it-rains-it-pours/">When it rains, it pours</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/09/pro-sql-server-2008-service-broker/">Pro SQL Server 2008 Service Broker</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/11/01/remove-pooling-for-data-changes-from-a-wcf-front-end/">Remove pooling for data changes from a WCF front end</a> <span class="post-date">2007 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/10/31/error-handling-in-service-broker-procedures/">Error Handling in Service Broker procedures</a> <span class="post-date">2007 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/08/21/service-broker-leaked-target-conversation-endpoints-fix-ships-in-cumulative-update-for-sql-server-2005-sp2/">Service Broker 'leaked' target conversation endpoints fix ships in Cumulative Update for SQL Server 2005 SP2</a> <span class="post-date">2007 Aug</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/07/31/dynamic-routing-service/">Dynamic Routing Service</a> <span class="post-date">2007 Jul</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/05/03/recycling-conversations/">Recycling Conversations</a> <span class="post-date">2007 May</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/04/25/reusing-conversations/">Reusing Conversations</a> <span class="post-date">2007 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/04/11/consuming-event-notifications-from-clr/">Consuming Event Notifications from CLR</a> <span class="post-date">2007 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2007/04/03/orlando-slides-and-code/">Orlando slides and code</a> <span class="post-date">2007 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/11/09/fast-way-to-check-message-count/">Fast way to check message count</a> <span class="post-date">2006 Nov</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/10/29/parallel-activation/">Parallel Activation</a> <span class="post-date">2006 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/10/16/writing-service-broker-procedures/">Writing Service Broker Procedures</a> <span class="post-date">2006 Oct</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/06/17/the-mysterious-notification/">The Mysterious Notification</a> <span class="post-date">2006 Jun</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/">Fire and Forget: Good for the military, but not for Service Broker conversations</a> <span class="post-date">2006 Apr</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/03/28/processing-conversation-with-priority-order/">Processing conversation with priority order</a> <span class="post-date">2006 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/03/07/call-a-procedure-in-another-database-from-an-activated-procedure/">Call a procedure in another database from an activated procedure</a> <span class="post-date">2006 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/03/01/signing-an-activated-procedure/">Signing an activated procedure</a> <span class="post-date">2006 Mar</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/01/30/how-long-should-i-expect-alter-databse-set-enable_broker-to-run/">How long should I expect ALTER DATABASE ... SET ENABLE_BROKER to run?</a> <span class="post-date">2006 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2006/01/12/why-does-feature-not-work-under-activation/">Why does feature &#8230; not work under activation?</a> <span class="post-date">2006 Jan</span>
    </li>
    <li>
      <a href="http://rusanu.com/2005/12/20/troubleshooting-dialogs/">Troubleshooting Dialogs</a> <span class="post-date">2005 Dec</span>
    </li>
  </ul>
</div>