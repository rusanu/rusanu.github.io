---
id: 869
title: SqlDependency based caching of LINQ Queries
date: 2010-08-02T15:23:04+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/08/02/863-revision-6/
permalink: /2010/08/02/863-revision-6/
---
Query Notifications is the SQL Server feature that allows a client to subscribe to notifications that are send when data in the database changes _irrelevant of how that change occurs_. I have talked before about how Query Notifications work in the article [The Mysterious Notification](http://rusanu.com/2006/06/17/the-mysterious-notification/). This feature was designed specifically for client side cache invalidation: the applications runs a query, gets back the result and stores it in the cache. Whenever the result is changed because data was updated, the application will be notified and it can invalidate the cached result.

Leveraging Query Notifications from the managed clients is very easy due to the dedicated SqlDependency class that takes care of a lot of the details needed to be set up in place in order to be able to receive these notifications.

# Leveraging SqlDependency from LINQ queries

There is no clear guidance from MSDN on how to mix these two technologies: Query Notifications and LINQ. There are a few in the community who have given hints on what has to be done, like this article [Using SQLDependency objects with LINQ](http://dunnry.com/blog/UsingSQLDependencyObjectsWithLINQ.aspx) by Ryan Dunn ([blog](http://dunnry.com/blog/)|[twitter](http://twitter.com/dunnry)).

My goal is to propose an easy to use extension method that can add SqlDependency based caching to any IQueryable<T>. Usage should be as simple as:

<code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
var queryTags = from t in ctx.Tags select t;
var tags = queryTags.AsCached("Tags");
foreach (Tag t in tags)
{
  ...
}
&lt;/pre>
&lt;p></code>

The first invocation should run the query and return the result, setting up a SqlDependency notification and also caching the result. Subsequent invocations should return the cached result, without hitting the database. Any change to the Tags table in my example should trigger the SqlDependency and invalidate the cache. Next invocation would again run the query and return the updated result, setting up a new SqlDependency notification and caching the new result.

# LinqToCache project

My solution is available as the [LinqToCache project](http://code.google.com/p/linqtocache/). To cache a LINQ query results and get active SqlDependency notifications when the data was changed, simply download the appropriate DLL for your target framework (.Net 3.5 or .Net 4.0) and add it as a reference to your project. Now any LINQ query (any IQueryable) will have a new extension method <tt>AsCached</tt>. This method returns an IEnumerable that of the query result. First invocation will always hit the database and set up a SqlDependency, subsequent invocations will return the cached result as long as it was not invalidated.

# Query Notifications restrictions

Not every query can be subscribed for notifications. The gory details of what works and what doesn&#8217;t are described in MSDN at <a href="http://msdn.microsoft.com/en-us/library/ms181122.aspx" target="_blank">Creating a Query for Notification</a>:

<quote>

  * The projected columns in the SELECT statement must be explicitly stated, and table names must be qualified with two-part names. Notice that this means that all tables referenced in the statement must be in the same database.
  * The statement may not use the asterisk (\*) or table_name.\* syntax to specify columns.
  * The statement may not use unnamed columns or duplicate column names.
  * The statement must reference a base table.
  * The statement must not reference tables with computed columns.
  * The projected columns in the SELECT statement may not contain aggregate expressions unless the statement uses a GROUP BY expression. When a GROUP BY expression is provided, the select list may contain the aggregate functions COUNT_BIG() or SUM(). However, SUM() may not be specified for a nullable column. The statement may not specify HAVING, CUBE, or ROLLUP.
  * A projected column in the SELECT statement that is used as a simple expression must not appear more than once.
  * The statement must not include PIVOT or UNPIVOT operators.
  * The statement must not include the UNION, INTERSECT, or EXCEPT operators.
  * The statement must not reference a view.
  * The statement must not contain any of the following: DISTINCT, COMPUTE or COMPUTE BY, or INTO.
  * The statement must not reference server global variables (@@variable_name).
  * The statement must not reference derived tables, temporary tables, or table variables.
  * The statement must not reference tables or views from other databases or servers.
  * The statement must not contain subqueries, outer joins, or self-joins.
  * The statement must not reference the large object types: text, ntext, and image.
  * The statement must not use the CONTAINS or FREETEXT full-text predicates.
  * The statement must not use rowset functions, including OPENROWSET and OPENQUERY.
  * The statement must not use any of the following aggregate functions: AVG, COUNT(*), MAX, MIN, STDEV, STDEVP, VAR, or VARP.
  * The statement must not use any nondeterministic functions, including ranking and windowing functions.
  * The statement must not contain user-defined aggregates.
  * The statement must not reference system tables or views, including catalog views and dynamic management views.
  * The statement must not include FOR BROWSE information.
  * The statement must not reference a queue.
  * The statement must not contain conditional statements that cannot change and cannot return results (for example, WHERE 1=0).
  * The statement can not specify READPAST locking hint.
  * The statement must not reference any Service Broker QUEUE.
  * The statement must not reference synonyms.
  * The statement must not have comparison or expression based on double/real data types.
  * The statement must not use the TOP expression.

</quote>

Although this list of restrictions is pretty severe, there is still room left for plenty of useful queries than can be cached using SqlDependency notifications for invalidation.

# Linq to SQL

Straight forward LINQ to SQL queries are valid for Query Notifications, as long as the first restriction listed above is cleared:  


> table names must be qualified with two-part names</p>
. In practice, this means simply fully qualifying the table names in the context designer, or in the [Table] attribute on the class. that is, always use &#8216;dbo.Table&#8217; instead of simply &#8216;Table&#8217; (of course, replace &#8216;dbo&#8217; with appropriate schema if necessary).