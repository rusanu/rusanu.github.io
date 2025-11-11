---
id: 63
title: Prevent ERRORLOG growth
date: 2007-11-11T01:21:36+00:00
author: remus
layout: post
guid: /2007/11/11/prevent-errorlog-growth/
permalink: /2007/11/11/prevent-errorlog-growth/
categories:
  - Troubleshooting
---
A problem I&#8217;ve often seen with many SQL Server deployments is when ERRORLOG grows and grows and grows, eventually filling all the available disk space. Reasons for this growth can be various, but the surprising thing is that sometimes they are absolutely legit. Kevin Farlee mentions on this blog entry <http://blogs.msdn.com/sqlserverstorageengine/archive/2007/10/30/when-is-too-much-success-a-bad-thing.aspx> about a site that is doing backups every 12 seconds, and each backup is reporting a success entry in ERRORLOG. Other reasons I&#8217;ve seen in production that caused growth were related to SqlDependency like the ones I mentioned previously here: <http://blogs.msdn.com/remusrusanu/archive/2007/10/12/when-it-rains-it-pours.aspx>. Now one thing to remember is that irellevant to what is causing the growth, there actualy is a stored procedure that forces an errorlog cycle: <a target="_blank" href="http://msdn2.microsoft.com/en-us/library/ms182512.aspx" title="sp_cycle_errorlog">sp_cycle_errorlog</a>. This procedure closes the current ERRORLOG, renames it as ERRORLOG.1 and opens a new one, pretty much like a server restart. If ERRORLOG growth is an issue in your environment, add an Agent job to periodically cycle the current ERRORLOG, perhaps followed up by an archive step.