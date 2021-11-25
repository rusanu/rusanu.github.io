---
id: 265
title: CLR Memory Leak
date: 2009-01-19T13:15:08+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/01/19/264-revision/
permalink: /2009/01/19/264-revision/
---
On a recent client engagement I had to investigate what appeared a to be a memory leak in a manager application. The program was running for about a week now and it appeared to slowly degrade in performance over time. Although it appeared healthy at only about 245 MB of memory used, I decided to investigate. The fastest way in my opinion to down leaks in a running environment is to attach Windbg and use SOS:

  * Download the appropriate release of the system debugger. Its location on the Microsoft Download site changes every now and then, but right now it is located at <a href="http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/installx86.Mspx</a> for 32 bit platforms or <a href="http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx" target="_blank">http://www.microsoft.com/whdc/devtools/debugging/install64bit.mspx</a> for 64 bit platforms.
  * Start Windbg and attach it to your application.
  * Load SOS: 
    <pre>.loadby sos mscorks</pre>