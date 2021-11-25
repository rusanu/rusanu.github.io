---
id: 1260
title: Online Index Operations for indexes containing LOB columns
date: 2011-08-04T14:37:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/08/04/1256-revision-4/
permalink: /2011/08/04/1256-revision-4/
---
SQL Server supports online index and table rebuild operations which allow for maintenance operations to occur w/o significant downtime. While a table is being rebuild the table is fully utilizable, it can be queried and updated. any updates done to the table while the online rebuild operation is occurring will be contained in the final rebuilt table. A detailed explanation on how these online rebuild operations work can be found in the <a href="http://technet.microsoft.com/en-us/library/cc966402.aspx" target="_blank">Online Indexing Operations in SQL Server 2005</a> white paper. But Online Index Build operations in SQL Server 2005, 2008 and 2008 R2 do not support tables that contain LOB columns, attempting to do so would trigger an error:

> <span style="color:red">Msg 2725, Level 16, State 2, Line &#8230;<br /> An online operation cannot be performed for index &#8216;&#8230;&#8217; because the index contains column &#8216;&#8230;&#8217; of data type text, ntext, image, varchar(max), nvarchar(max), varbinary(max), xml, or large CLR type. For a non-clustered index, the column could be an include column of the index. For a clustered index, the column could be any column of the table. If DROP_EXISTING is used, the column could be part of a new or old index. The operation must be performed offline.</span> 

To be accurate the restriction applies not to tables, but to any index or heap that contains an LOB column. That, of course, includes any clustered index or the base heap of a table if the table contains any LOB columns, but it would include any non-clustered index that includes a LOB column. In other words I can rebuild online non-clustered indexes of any table as long as they don&#8217;t use the INCLUDE clause to add a LOB column from the base table, but for sure I cannot rebuild online the table itself (meaning the clustered index or the heap).

Starting with SQL Server 11 it is actually permitted to rebuild indexes and heaps containing LOB columns. All restrictions