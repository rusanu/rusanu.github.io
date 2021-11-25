---
id: 1724
title: How to enable Selective XML indexes in SQL Server 2012 SP1
date: 2013-02-11T02:41:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/02/11/1716-autosave/
permalink: /2013/02/11/1716-autosave/
---
SQL Server 2012 SP1 has shipped a great enhancement to XML: [Selective XML Indexes](http://msdn.microsoft.com/en-us/library/jj670108.aspx). When properly used these indexes can speed up the searching of XML columns tremendously, at little disk/size cost:

> The selective XML index feature lets you promote only certain paths from the XML documents to index. At index creation time, these paths are evaluated, and the nodes that they point to are shredded and stored inside a relational table in SQL Server. This feature uses an efficient mapping algorithm developed by Microsoft Research in collaboration with the SQL Server product team. This algorithm maps the XML nodes to a single relational table, and achieves exceptional performance while requiring only modest storage space.

However, at the time of this post, the documentation on MSDN omits a crucial requirement: the database has to be enabled to support these new indexes. Because is an on-disk format change shipped in a service pack release the engine cannot use the new selective XML indexes unless explicitly allowed. And the database version must change to a version that the RTM SQL Server 2012 does not recognize so that the database is not accidentally attached/restored on an RTM instance of SQL Sever 2012 that would not comprehend the new Selective XML Index objects and would panic (undefined behavior). To enable Selective XML Indexes in the database you must run [<tt>sp_db_selective_xml_index</tt>](http://msdn.microsoft.com/en-us/library/jj670102.aspx):

> Enables and disables Selective XML Index functionality on a SQL Server database. If called without any parameters, the stored procedure returns 1 if the Selective XML Index is enabled on a particular database.

<code class="prettyprint lang-sql">&lt;/pre>
&lt;p>EXECUTE sys.sp_db_selective_xml_index&lt;br />
    @db_name = N'AdventureWorks2012'&lt;br />
  , @action = N'true';&lt;br />
GO
&lt;/pre>
&lt;p></code>

Be aware that once upgraded to support Selective XML indexes this database can no longer be attached or restored on an RTM instance of SQL Server 2012. This applies to log shipping, Database Mirroring and AlwaysOn relationships, which will break when this upgrade is performed. _If_ all the partners in the log shipping, DBM session or AG are also upgraded to SQL Server 2012 SP1 you can re-enable the relationship **after** you enabled Selective XML Indexes.