---
id: 2146
title: How to analyse SQL Server performance
date: 2014-01-07T02:54:41+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/01/07/2144-revision-2/
permalink: /2014/01/07/2144-revision-2/
---
So you have this SQL Server database that your application uses and it somehow seems to be slow. How do you troubleshoot this problem? Where do you look? What do you measure? I will try to give redux of my know-how on this topic. I hope this article will answer enough questions to get you started so that you can identify the bottlenecks yourself, or know what to search for to further extend your arsenal and knowledge.

# How does SQL Server work

to be able to troubleshoot performance problems I think that you need to have an understanding of how SQL Server works. I have an detailed explanation article at [Understanding how SQL Server executes a query](http://rusanu.com/2013/08/01/understanding-how-sql-server-executes-a-query/).