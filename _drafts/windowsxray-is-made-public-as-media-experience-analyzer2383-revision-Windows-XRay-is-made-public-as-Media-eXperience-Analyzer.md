---
id: 2384
title: Windows XRay is made public as Media eXperience Analyzer
date: 2014-05-30T00:06:59+00:00
author: remus
layout: revision
guid: http://rusanu.com/2014/05/30/2383-revision/
permalink: /2014/05/30/2383-revision/
---
One of the coolest performance tools I used internally at Microsoft was the WindowsXRay. This is a tools that can analyze ETW data and offer exquisite visualizations, similar to [Windows Performance Analyzer (WPA)](http://msdn.microsoft.com/en-us/library/windows/hardware/hh448170.aspx). I&#8217;m excited that Microsoft decided to release the WindowsXRay under the public moniker of [Media eXperience Analyzer](http://www.microsoft.com/en-us/download/details.aspx?id=43105). The tool was internally developed by the multimedia team, so the release is naturally geared and marketed toward the multimedia developers. But rest assured, the tool can be used for all sort of performance analysis and I had successfully used in analysis of SQL Server performance problems.

Using the Media eXperience Analyzer (XA) is similar to the WPA experience: you typically need to collect an ETL trace using the platform ETL tools like the Windows Performance Recorder (WPR)