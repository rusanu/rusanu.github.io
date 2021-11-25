---
id: 2532
title: '>Net Stopwatch suffers from cross CPU frequency inconsistencies'
date: 2017-10-19T00:31:42+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=2532
permalink: /?p=2532
categories:
  - Uncategorized
---
I recently was investigating performance problems in a .Net application (C#) and the code had used a [<tt>Stopwatch</tt>](https://msdn.microsoft.com/en-us/library/system.diagnostics.stopwatch(v=vs.110).aspx) to measure some calls. The function was using <tt>async<tt>/<tt>await</tt> so the <tt>Stopwatch</tt> would be started on one thread and stopped on another one. To my dismay, the function was reporting constantly duration of 14-15 seconds, when the entire test lasted under 10 seconds.</p>