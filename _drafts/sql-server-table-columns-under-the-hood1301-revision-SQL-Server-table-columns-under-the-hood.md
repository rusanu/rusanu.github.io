---
id: 1302
title: SQL Server table columns under the hood
date: 2011-09-24T16:30:39+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/09/24/1301-revision/
permalink: /2011/09/24/1301-revision/
---
You probably can easily answer a question like &#8216;What columns does this table have?&#8217;. Whether you use the SSMS object explorer, or <tt>sp_help</tt>, or you query <tt>sys.column</tt>, the answer is fairly easy to find. But what is I ask &#8216;What are the **physical** columns of this table?&#8217;. Huh? Is there any difference? Lets see.

At the logical layer tables have exactly the structure you declare it in your CREATE TABLE statement, and perhaps modifications from ALTER TABLE statements. This is the layer at which you can look into <tt>sys.columns</tt> and see the table structure, or look at the table in SSMS object explorer and so on and so forth. But there is also a lower layer, the physical layer of the storage engine where the table might have surprisingly different structure from what you expect.

### Inspecting the physical table structure

To view the physical table structure you must use the undocumented system internals views: [<tt>sys.system_internals_partitions</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx) and [<tt>sys.system_internals_partition_columns</tt>](http://msdn.microsoft.com/en-us/library/ms189600.aspx):


<code class="prettyprint lang-sql">
&lt;pre>
&lt;/pre>
&lt;p></code>