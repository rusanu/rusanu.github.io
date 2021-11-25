---
id: 2135
title: SQL Server clustered columnstore Tuple Mover
date: 2013-12-02T04:49:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/12/02/2123-revision-12/
permalink: /2013/12/02/2123-revision-12/
---
The [updateable clustered columnstore indexes introduced with SQL Server 2014](http://rusanu.com/2013/06/11/sql-server-clustered-columnstore-indexes-at-teched-2013/) rely on a background task called the <tt>Tuple Mover</tt> to periodically compress deltastores into the more efficient columnar format. Deltastores store data in the traditional row-mode (they are B-Trees) and as such are significantly more expensive to query that the compressed columnar segments. How more expensive? They are equivalent to storing the data in an uncompressed Heap and, due to small size (max 1048576 rows per deltastore rowset), they get little traction from parallelism and from read-aheads. It is important for your upload, initial seed and day-to-day ETL activities to achieve a healthy columnstore index, meaning no deltastores or only a few deltastores. To achieve this desired state of a healthy columnstore it is of paramount importance to understand how deltastores are created and removed.

<!--more-->

## How are Deltastores created

Deltastores appear through one of the following events:

  * Trickle <tt>INSERT</tt>: ordinary INSERT statements that do not use the BULK INSERT API. That includes all <tt>INSERT</tt> statements except <tt>INSERT ... SELECT ...</tt>.
  * <tt>UPDATE</tt>, <tt>MERGE</tt> statements: all updates are implemented in clustered columnstores as a delete of old data and insert of new (modified) data. The insert of new data as result of data updates will always be a trickle insert.
  * **Undersized** BULK INSERT operations: insufficient number of rows of data inserted using the BULK INSERT API (that is using one of [<tt>IRowsetFastLoad</tt>](http://technet.microsoft.com/en-us/library/ms131708.aspx), use the [SQL Native client ODBC Bulk operation extensions](http://msdn.microsoft.com/en-us/library/ms130792.aspx) or the .Net managed [<tt>SqlBulkCopy</tt>](http://msdn.microsoft.com/en-us/library/system.data.sqlclient.sqlbulkcopy.aspx) class) and <tt>INSERT ... SELECT ...</tt> statements</tt>. These BULK INSERT APIs are often better known by the various tools that expose them, like <tt>bcp.exe</tt> or the &#8216;fast-load&#8217; SSIS OleDB destination. _Insufficient_ rows means that the number of rows submitted **in a batch** is too small to create a compressed segment **in a partition**. The exact number is, I think, around 100000 rows. Bellow this margin, even when BULK INSERT API is used, the rows will be inserted as a deltastore. Above 100000 a compressed segment is created, but do note that a clustered columnstore consisting of 100k rows segments will be sub-optimal. The ideal batch size is 1 million rows, and your ETL process should thrive to achieve this batch size **per partition per thread** (ie each inserting thread should insert 1 million rows for each partition it inserts into).

Of course, the business process may simply not have enough data in an ETL pass to create the desired 1 million rows batch size, or even the minimum 100k rows. That is fine, rows will be saved as deltastores and the deltastores will be later compressed, when they fill up (reach the max size of 1048576 rows). What is important though is that during **initial population** (seeding) the upload process **must** create healthy clustered columnstores. The best option is to upload the data into a row-mode structure (heap or a b-tree) and then issue a <tt>CREATE CLUSTERED COLUMNSTORE INDEX</tt> statement to build the clustered columnstore. This option is the best because only CREATE INDEX on existing data will create appropriately relevant (ie. useful) global dictionaries for the compressed segments.Having good global dictionaries reduces the size of compressed segments significantly, for improved query performance. If you&#8217;re afraid to run <tt>CREATE CLUSTERED COLUMNSTORE INDEX</tt> on a large data set due to possible log growth issues, remember two things: CREATE INDEX is a prime candidate for minimally logging and a clustered columnstore index will **not** generate a huge row-by-row log even in fully logged mode, because it only writes compressed data.

<p class="callout float-right">
  Global dictionaries are created only during index build
</p>

If you cannot upload the initial data in row-mode and then create the clustered index, the alternative is to BULK INSERT into an empty clustered columnstore. This will result in a worse quality columnstore because it cannot use global dictionaries, only local dictionaries can be used. This, in general, results in worse compression. Of course the compression will still be order of magnitude better than row-mode, but will probably not be as good as a fresh index build could achieve. But the bigger problem you&#8217;ll have with initial population of an empty clustered columnstore will be achieving the desired 1 million rows per partition per thread batch size. Think what that means in terms of an SSIS [<tt>DefaultBufferSize</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.dts.pipeline.wrapper.mainpipeclass.defaultbuffersize(v=sql.105).aspx) and [<tt>DefaultBufferMaxRow</tt>](http://msdn.microsoft.com/en-us/library/microsoft.sqlserver.dts.pipeline.wrapper.mainpipeclass.defaultbuffermaxrows(v=sql.105).aspx)!

The absolutely worse option for initial population of data is to upload it in small batches into an existing columnstore index, let it settle into deltastores let the Tuple Mover compress it.

## How are Deltatsores compressed

When a deltastore fills up (ie. it reaches the max size of 1048576 rows) is going to be closed and will become available for the Tuple Mover to compress it. The Tuple Mover will create big, healthy segments, true. But it is not designed to be a replacement for index build. What does that mean?

    
    create event session cci_tuple_mover
    on server
     add event sqlserver.columnstore_tuple_mover_begin_compress,
     add event sqlserver.columnstore_tuple_mover_end_compress
     add target package0.asynchronous_file_target
       (SET filename = 'c:\temp\cci_tuple_mover.xel', metadatafile = 'c:\temp\cci_tuple_mover.xem');
    

The Tuple Mover doesn&#8217;t expose much with regard to monitoring, but it does expose two XEvents for when it starts to compress a segment and when it completes a segment compression. When I create  
a couple of ready deltastores and wait for the Tuple Mover to compress them, this is what I see in the XEvents session:

    
    with e as (select cast(event_data as xml) as x 
    	from sys.fn_xe_file_target_read_file ('c:\temp\*.xel', 'c:\temp\*.xem', null, null))
    select x.value(N'(//event/@name)[1]', 'varchar(40)') as event,
    	x.value(N'(//event/@timestamp)[1]', 'datetime') as time
    	from e
    	order by time desc;
    
    event                                    time
    ---------------------------------------- -----------------------
    columnstore_tuple_mover_end_compress     2013-12-02 11:26:25.047
    columnstore_tuple_mover_begin_compress   2013-12-02 11:26:20.247
    columnstore_tuple_mover_end_compress     2013-12-02 11:26:05.040
    columnstore_tuple_mover_begin_compress   2013-12-02 11:26:00.183
    

You can see that the Tuple Mover takes about 5 seconds to compress a row group, which is the ballpark time span you should be expecting too. Wider rows (many columns) will take more. It cannot benefit from parallelism. It is also important to notice that the Tuple Mover does nothing for about 15 seconds between the two rowgroups it compressed. This is not waiting for some resource, is literally how it works: it compresses one rowgroup at a time per table, and then it sleeps. In addition, the Tuple Mover only wakes up and looks for &#8216;work&#8217; every few minutes.

As you can see, the Tuple Mover is not trying to race to keep up with a massive bulk insert. For such situations, the ETL pipeline should create the rowgroups straight into compressed form, or you should use index build operations. Tuple Mover works at a pace that is designed to keep up with a normal rhythm of business trickle INSERT, restricting itself to minimal resource consumption. If you do operations that end up with a massive number of deltastores, the Tuple Mover can catch up at a pace of 5-6 segments per minute, maybe even less. If your index has hundreds or even thousands of CLOSED deltastores then you should consider an index REBUILD operation.

BTW, did you know the columnstore index offline rebuild operations are actually semi-online? They only acquire S locks on the index during the rebuild phase and then escalate to a short SCH-M lock at the end, when they swap the old and new index. Queries can read the old index during the rebuild, but no updates/deletes/inserts are allowed.