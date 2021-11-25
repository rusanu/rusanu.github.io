---
id: 2370
title: Hive introduction for SQL Server professionals
date: 2014-04-09T00:38:07+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/04/09/2340-revision-6/
permalink: /2014/04/09/2340-revision-6/
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

**Hadoop MapReduce** is first and foremost a framework for running programs in a distributed cluster of execution machines. Programs as submitted as &#8216;jobs&#8217; and the program is responsible into splitting its execution into a series of tasks, each task being independently executed on one of the available machines in the cluster. The MapReduce framework handles this execution by offering the following:

Scheduling
:   The framework handles the scheduling of execution of the task on the available nodes.

Binaries and Configuration Distribution
:   Tasks require the code to be distributed to the executing nodes, along with any additional libraries and configuration files required during execution.

Collecting Results
:   

Failures and Retries
:   Tasks that fail to execute or time out due to a variety of reasons are automatically retried by the MapReduce framework.

Cluster Health
:   Nodes that repeatedly fail execution or nodes that lost contact with the cluster are automatically blacklisted (marked as unavailable for execution) and the framework handles the rebalancing of pending tasks in the remaining &#8216;good&#8217; cluster.

Collecting Execution Logs
: