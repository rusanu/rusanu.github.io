---
id: 864
title: SqlDependency based caching of LINQ Queries
date: 2010-08-02T14:41:28+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/08/02/863-revision/
permalink: /2010/08/02/863-revision/
---
Query Notifications is the SQL Server feature that allows a client to subscribe to notifications that are send when data in the database changes _irrelevant of how that change occurs_. I have talked before about how Query Notifications work in the article The Mysterious Notification</i>. This feature was designed specifically for client side cache invalidation: the applications runs a query, gets back the result and stores it in the cache. Whenever the result is changed because data was updated, the application will be notified and it can invalidate the cached result.</p> 

Leveraging Query Notifications from the managed clients is very easy due to the dedicated SqlDependency class that takes care of a lot of the details needed to be set up in place in order to be able to receive these notifications.

# Leveraging SqlDependency from LINQ queries

There is no clear guidance from MSDN on how to mix these two technologies: Query Notifications and LINQ. There are a few in the community who have given hints on what has to be done, like this article [Using SQLDependency objects with LINQ](http://dunnry.com/blog/UsingSQLDependencyObjectsWithLINQ.aspx) by Ryan Dunn ([blog](http://dunnry.com/blog/))