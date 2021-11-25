---
id: 866
title: SqlDependency based caching of LINQ Queries
date: 2010-08-02T14:54:44+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/08/02/863-revision-3/
permalink: /2010/08/02/863-revision-3/
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