---
id: 1257
title: Online Index Build for tables with LOB columns
date: 2011-08-04T14:08:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/04/1256-revision/
permalink: /2011/08/04/1256-revision/
---
Online Index Build operations in SQL Server 2005, 2008 and 2008 R2 does not support tables that contain LOB columns, attempting to do so would trigger an error:

> <span style="color:red">Msg 2725, Level 16, State 2, Line &#8230;<br /> An online operation cannot be performed for index &#8216;&#8230;&#8217; because the index contains column &#8216;&#8230;&#8217; of data type text, ntext, image, varchar(max), nvarchar(max), varbinary(max), xml, or large CLR type. For a non-clustered index, the column could be an include column of the index. For a clustered index, the column could be any column of the table. If DROP_EXISTING is used, the column could be part of a new or old index. The operation must be performed offline.</span>