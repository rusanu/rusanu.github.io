---
id: 2388
title: WindowsXRay is made public as Media eXperience Analyzer
date: 2014-05-30T04:33:25+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/05/30/2383-revision-5/
permalink: /2014/05/30/2383-revision-5/
---
One of the most useful performance tools I used internally at Microsoft was the WindowsXRay. I&#8217;m pleased to find that it was released the new name of [Media eXperience Analyzer](http://www.microsoft.com/en-us/download/details.aspx?id=43105). The tool was internally developed by the Windows media perf team and the release info is targeted toward the media developers. But rest assured, the tool can be used for all sort of performance analysis and I had successfully used in analysis of SQL Server performance problems. Using the Media eXperience Analyzer (XA) you typically start by collecting one or more ETL traces using the platform tools like the [Windows Performance Recorder (WPR)](http://msdn.microsoft.com/en-us/library/windows/hardware/hh448205.aspx). XA is a visualization tool used to inspect these ETL traces, much like WPA is.