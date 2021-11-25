---
id: 725
title: Performance comparison of varchar(max) vs. varchar(N)
date: 2010-03-17T22:36:25+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/17/724-revision/
permalink: /2010/03/17/724-revision/
---
The question of comparing the MAX types (VARCHAR, NVARCHAR, VARBINARY) with their non-max counterparts is often asked, but the answer usually gravitate around the storage differences. But I&#8217;d like to address the point that these types have inherent, intrinsic performance differences that are not driven by different storage characteristics. In other words, simply comparing and manipulating variables and columns in T-SQL can yield different performance when VARCHAR(MAX) is used vs. VARCHAR(N).