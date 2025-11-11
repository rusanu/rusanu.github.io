---
id: 216
title: High Performance Windows programs
date: 2008-11-11T10:48:28+00:00
author: robert
layout: post
guid: /?p=216
permalink: /2008/11/11/high-performance-windows-programs/
categories:
  - Announcements
---
Recently I wanted to go over again Rick Vicik papers on high performance programs on the Windows platform. These papers are a true Bible for anyone in need to write truly highly scalable and high performance server applications. They address the back end C/C++ programming and explain how to properly use the Windows threading, optimize I/O and specially the importance of data cache conscious programming, NUMA object allocations and access locality and impact of data sharing on performance. I do find however that many of the principles explained there apply just as well to C# and .Net programming. I wanted to refresh my memory on some issues so I searched for them and to my delight I found that Rick updated the papers for Vista and Windows 2008 and had posted them as a three part series on the <a href="http://blogs.technet.com/winserverperformance" target="_blank">Windows Performance blog</a> and I wanted to share these with my blog audience:

  * <a href="http://blogs.technet.com/winserverperformance/archive/2008/04/25/designing-applications-for-high-performance-part-1.aspx" target="_blank">Designing Applications for High Performance &#8211; Part I</a>
  * <a href="http://blogs.technet.com/winserverperformance/archive/2008/05/21/designing-applications-for-high-performance-part-ii.aspx" target="_blank">Designing Applications for High Performance &#8211; Part II</a>
  * <a href="http://blogs.technet.com/winserverperformance/archive/2008/06/26/designing-applications-for-high-performance-part-iii.aspx" target="_blank">Designing Applications for High Performance &#8211; Part III</a>