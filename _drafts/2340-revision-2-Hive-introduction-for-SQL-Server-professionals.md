---
id: 2365
title: Hive introduction for SQL Server professionals
date: 2014-04-09T00:12:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/04/09/2340-revision-2/
permalink: /2014/04/09/2340-revision-2/
---
Over the past year I&#8217;ve been contributing to the Apache Hive project and I got to know this product, as well as Hadoop. In the SQL Server community I know that Hive is generally unknown so I thought it would be an interesting topic to discuss how Hive works, and specially compare with how SQL Server works. My experience with both SQL Server and Hive is that of a contributor. This is a different point of view from how an end user (developer or admin) would probably compare the two.

Apache Hive is a data warehouse oriented SQL database that uses Hadoop as an execution engine. It has started as a project inside Facebook and is now a very active Apache Foundation top-level project with contributions from many big companies. Hive strengths are:

  * leverage existing SQL language know-how
  * processing large and very large data sets
  * processing data that is already resident in Hadoop HDFS storage
  * leverage hardware of Hadoop clusters
  * processing a variety of file formats natively
  * extensible to accommodate custom file formats
  * extensible to accommodate custom UDFs
  * free pricing, no licensing restrictions

<!--more-->

# What is Hadoop and how does it work

In order to understand Hive one must have an idea how Hadoop works so I feel obligated to do a short detaour to pre