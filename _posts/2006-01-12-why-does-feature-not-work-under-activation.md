---
id: 22
title: 'Why does feature &#8230; not work under activation?'
date: 2006-01-12T09:14:08+00:00
author: remus
layout: post
guid: http://rusanu.com/2006/01/12/why-does-feature-not-work-under-activation/
permalink: /2006/01/12/why-does-feature-not-work-under-activation/
categories:
  - Troubleshooting
tags:
  - activation
  - error
  - security
  - service broker
---
A number of features behave differently when running under activated stored proc. Selecting from server level views, selecting from dynamic management views, using linked servers and other. Executing the same procedure from a user session yields different results. E.g. if the procedures looks up a dynamic management view like sys.dm_&#8230;, it gets fewer rows when running under activation than when running from a user session.

What happens is that you overlook the fact that activation is executing under an EXECUTE AS context. The BOL have a great article explaining the behavior of EXECUTE AS, &#8216;Extending Database Impersonation by Using EXECUTE AS&#8217;, <http://msdn2.microsoft.com/en-us/library/ms188304.aspx>. It is not activation that causes the different behavior, but the fact that activation ALWAYS uses an EXECUTE AS context. The &#8216;Troubleshooting Activated Stored Procedures&#8217; topic in BOL ([http://msdn2.microsoft.com/en-us/library/ms166102(en-US,SQL.90).aspx](http://msdn2.microsoft.com/en-us/library/ms166102%28en-US,SQL.90%29.aspx)) actually explains this and recommends that you also use an EXECUTE AS in the user session when invoking the procedure for debugging purposes, to get the same behavior and results as when activated.

A short explanation of the problem is that the activation execution context is trusted only in the database, not in the whole server. Anything related to the whole server, like a server level view or a dynamic management view or a linked server, acts as if you logged in as [Public].

The recommended way to solve the issue is to sign the procedure with a server level certificate that has the proper rights needed for the operation in question (see <http://msdn2.microsoft.com/en-us/library/ms181700.aspx>). Another possible way to fix the problem is to mark the database as TRUSTWORTHY, using ALTER DATABASE. But you better read the carefully the &#8216;Extending Database Impersonation by Using EXECUTE AS&#8217; before doing this step and make sure you understand all the implications. You should do this only if you completely trust the dba of the database in question, as he becomes a de facto sysadmin when the database is marked as trustworthy.