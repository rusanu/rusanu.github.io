---
id: 2371
title: Hive introduction for SQL Server professionals
date: 2014-04-09T01:36:02+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/04/09/2340-revision-7/
permalink: /2014/04/09/2340-revision-7/
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

In order to understand Hive one must have an idea how Hadoop works so I feel obligated to do a short detour to explain the basics. The term &#8220;Hadoop&#8221; is used loosely today to refer to an entire ecosystem of products that run on Hadoop like [HBase](https://hbase.apache.org/), [Pig](https://pig.apache.org/), [Oozie](https://oozie.apache.org/), [Sqoop](https://sqoop.apache.org/), [Mahout](https://mahout.apache.org/) and many more, including [Hive](https://hive.apache.org/). But in a more strict sense Hadoop refers to two core products: HDFS and MapReduce.

**Hadoop MapReduce** is first and foremost a framework for running programs in a distributed cluster of execution machines. It handles scheduling of execution on the available machines in the cluster, retries of failed or timed out tasks, distribution of execution code and configuration files to the machines tasked with executing that code, collecting execution logs, maintaining cluster health by blacklisting machines that repeatedly fail execution and so on and so forth. Programs submitted for execution must be implemented as a map-reduce program. A map-reduce program splits its execution into a **map** phase where it operates on random input that produces output that can be sorted by a key and a **reduce** phase that operates on sorted input. Most algorithms can be rewritten in form of a map-reduce program and this gives them inherent parallelizability: the map phase can be split into many tasks that operated in parallel on parts of a big input and the reduce phase can implement efficient computation by knowing that it operates on sorted input. The MapReduce framework handles reading of the input files, as well as the sorting and grouping of the map output (the &#8216;shuffle&#8217; phase) and then finally the reading of the sorted intermediate map output into the reduce phase. This leaves the program with the much simpler code of reading one &#8216;row&#8217; at a time and simply &#8217;emit&#8217; an output row, both during the map phase and during the reduce phase.

[<img src="http://rusanu.com/wp-content/uploads/2014/04/hadoop-map-reduce.png" alt="" title="hadoop-map-reduce" width="600" class="alignleft size-full wp-image-2369" />](http://rusanu.com/wp-content/uploads/2014/04/hadoop-map-reduce.png)