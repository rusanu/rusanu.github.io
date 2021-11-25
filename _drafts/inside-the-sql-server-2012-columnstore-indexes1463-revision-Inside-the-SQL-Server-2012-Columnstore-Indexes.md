---
id: 1464
title: Inside the SQL Server 2012 Columnstore Indexes
date: 2012-05-16T02:58:48+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/05/16/1463-revision/
permalink: /2012/05/16/1463-revision/
---
Columnar storage has established itself as the de-facto option for Business Intelligence (BI) storage. The traditional row-oriented storage of RDBMS was designed for fast single-row oriented OLTP workloads and it has problems handling the large volume range oriented analytical processing that characterizes BI workloads. But what _is_ columnar storage and, more specifically, how does SQL Server 2012 implement columnar storage with the new <tt>COLUMNSTORE</tt> indexes?

The defining characteristic of columnar storage is the ability to read the values of a particular column of a table without having to read the values of all the other columns. In row-oriented storage this is impossible because the individual column values are physically stored grouped in rows on the pages and reading a pa

<!--more-->