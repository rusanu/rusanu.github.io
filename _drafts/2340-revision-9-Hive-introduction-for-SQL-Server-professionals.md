---
id: 2373
title: Hive introduction for SQL Server professionals
date: 2014-04-09T04:39:40+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/04/09/2340-revision-9/
permalink: /2014/04/09/2340-revision-9/
---
Over the past year I&#8217;ve been contributing to the Apache Hive project and I got to know this product, as well as Hadoop. In the SQL Server community I know that Hive is generally unknown so I thought it would be an interesting topic to discuss how Hive works, and specially compare with how SQL Server works. My experience with both SQL Server and Hive is that of a contributor. This is a different point of view from how an end user (developer or admin) would probably compare the two.

Apache Hive is a data warehouse oriented SQL database that uses Hadoop as an execution engine. It has started as a project inside Facebook and is now a very active Apache Foundation top-level project with contributions from many big companies. Hive strengths are:

  * leverage existing SQL language know-how
  * processing large and very large data sets
  * directly access data resident in Hadoop HDFS storage
  * leverage hardware of Hadoop clusters
  * processing a variety of file formats natively
  * extensible to accommodate custom file formats
  * extensible to accommodate custom UDFs
  * free pricing, no licensing restrictions

<!--more-->

# What is Hadoop and how does it work

In order to understand Hive one must have an idea how Hadoop works so I feel obligated to do a short detour to explain the basics. The term &#8220;Hadoop&#8221; is used loosely today to refer to an entire ecosystem of products that run on Hadoop like [HBase](https://hbase.apache.org/), [Pig](https://pig.apache.org/), [Oozie](https://oozie.apache.org/), [Sqoop](https://sqoop.apache.org/), [Mahout](https://mahout.apache.org/) and many more, including [Hive](https://hive.apache.org/). But in a more strict sense Hadoop refers to two core products: HDFS and MapReduce. For simplicity I will ignore [YARN](http://hadoop.apache.org/docs/r2.3.0/hadoop-yarn/hadoop-yarn-site/YARN.html) for this discussion.

**Hadoop [MapReduce](http://en.wikipedia.org/wiki/MapReduce)** is first and foremost a framework for running programs in a distributed cluster of execution machines. It handles scheduling of execution on the available machines in the cluster, retries of failed or timed out tasks, distribution of execution code and configuration files to the machines tasked with executing that code, collecting execution logs, maintaining cluster health by blacklisting machines that repeatedly fail execution and so on and so forth. Programs submitted for execution must be implemented as a map-reduce program. A map-reduce program splits its execution into a **map** phase where it operates on random input that produces output that can be sorted by a key and a **reduce** phase that operates on sorted input. Most algorithms can be rewritten in form of a map-reduce program and this gives them inherent parallelizability: the map phase can be split into many tasks that operated in parallel on parts of a big input and the reduce phase can implement efficient computation by knowing that it operates on sorted input. The MapReduce framework handles reading of the input files, as well as the sorting and grouping of the map output (the &#8216;shuffle&#8217; phase) and then finally the reading of the sorted intermediate map output into the reduce phase. This leaves the program with the much simpler code of reading one &#8216;row&#8217; at a time and simply &#8217;emit&#8217; an output row, both during the map phase and during the reduce phase.

[<img src="http://rusanu.com/wp-content/uploads/2014/04/hadoop-map-reduce.png" alt="" title="hadoop-map-reduce" width="600" class="alignleft size-full wp-image-2369" />](http://rusanu.com/wp-content/uploads/2014/04/hadoop-map-reduce.png)

**[HDFS](http://en.wikipedia.org/wiki/Apache_Hadoop#File_system)** is a distributed file system optimized for usage in Hadoop clusters. It&#8217;s most important characteristic is that it splits the files into **blocks** (usually of some 64-256 MB size, configurable). Blocks are stored on the cluster machines, usually having 3 copies of each block on 3 different machines (again, configurable via the &#8220;replication factor&#8221;). Loosing machines in the cluster or adding new machines results in new block copies (&#8220;replicas&#8221;) being created to keep the desired number of copies for each block and is handled automatically. One machine in the cluster runs the **namenode** service that is the &#8216;brain&#8217; of the file system, it keep tracks of the block locations and grants access to the files. HDFS handles very well adding new data into the file system (writing new files, appending data to a file). All Hadoop ecosystem applications take advantage of this and are coded in an &#8220;append only&#8221; fashion, they basically never update exiting files, only append more data as new files.

Together HDFS and MapReduce add synergy to the Hadoop cluster: MapReduce is aware of how HDFS stores the files and will schedule the execution of map tasks to occur on the same machine that hosts one of the blocks (replicas) of the input data (split) for that task. This way the processing occurs on the same location where the data is, avoiding network transfer. Together, MapReduce and HDFS push the execution where the data is. Bear in mind that you can use each of the two core Hadoop components separately, there are applications that use HDFS w/o making use of MapReduce and there are MapReduce clusters backed by different file systems, Azure HDInsight for instance [uses Azure Blob Storage instead of HDFS](http://azure.microsoft.com/en-us/documentation/articles/hdinsight-use-blob-storage/).

Finally **Hadoop Commons** are necessary libraries to build map-reduce programs that run on a Hadoop MapReduce cluster. These libraries offer the necessary code to build a program that submits a map-reduce program (a &#8216;job&#8217;) into a Hadoop cluster, the code to read the task&#8217;s input and produce the required output, and code to understand the various file formats common in Hadoop ecosystem (eg. [sequence files](http://wiki.apache.org/hadoop/SequenceFile)), code to access HDFS as a filesystem (not through a map-reduce program) and so on and so forth.

# How does Hive use Hadoop</p>