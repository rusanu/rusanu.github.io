---
id: 515
title: Asynchronous procedure execution
date: 2009-08-04T23:53:55+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/08/04/514-revision/
permalink: /2009/08/04/514-revision/
---
Recently an user on StackOverflow raised the question <a href="http://stackoverflow.com/questions/1229438/execute-a-stored-procedure-from-a-windows-form-asynchronously-and-then-disconnect" target="_blank">Execute a stored procedure from a windows form asynchronously and then disconnect?</a>. This is a known problem, how to invoke a long running procedure on SQL Server without constraining the client to wait for the procedure execution to terminate. Most times I&#8217;ve seen this question raised in the context of web applications when waiting for a result means delaying the response to the client browser. On Web apps the time constraint is even more drastic, the developer often desires to launch the procedure and immediately return the page, having to retrieve the execution result later, usually via an Ajax call driven by the returned page script.

Frankly I was a bit surprised to see that the responses gravitated either around the SqlClient asynchronous methods (BeginExecute&#8230;) or around having a dedicated process with the sole pupose of maintaining the client connection alive for the duration of the long running procedure.

This problem is perfectly addressed by Service Broker Activation. Since I wanted to preserve the solution for further reference, I decided to put it in as a blog entry, with additional comments. For many of you Service Broker aficionados that read my blog regularly, this article is not inovatia rehash of well known techniques.