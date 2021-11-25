---
id: 1203
title: How to update a table with a columnstore index
date: 2011-07-13T00:30:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/07/13/1202-revision/
permalink: /2011/07/13/1202-revision/
---
In my previous article [How to use columnstore indexes in SQL Server](http://rusanu.com/2011/07/13/how-to-use-columnstore-indexes-in-sql-server/) we&#8217;ve seen how to create a columnstore index on a table and how certain queries can significantly reduce the IO needed and thus increase in performance by leveraging this new feature. But once a columnstore index is added to a table, the table becomes read-only as it cannot be updated. Trying to insert a new a row in the table will result in an error:

<pre>insert into sales ([date],itemid, price, quantity) values ('20110713', 1,1.0,1);
</pre>