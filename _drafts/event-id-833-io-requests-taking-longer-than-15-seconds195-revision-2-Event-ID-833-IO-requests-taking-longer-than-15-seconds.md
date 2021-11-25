---
id: 197
title: 'Event ID 833: I/O requests taking longer than 15 seconds'
date: 2008-10-28T14:56:58+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/10/28/195-revision-2/
permalink: /2008/10/28/195-revision-2/
---
The error 833 is usually associated with hardware or system driver problems and the typical recommendation is to replace the hardware or update the drivers and firmware used. However there is a common scenario that leads to this problem when your hardware is fine and sound.

<!--more-->

The MSDN cause for error 833 is this: &#8220;<tt>This problem can be caused system performance issues, hardware errors, firmware errors, device driver problems, or filter driver intervention in the IO process.</tt>&#8220;. When you encounter this error in your ERRORLOG it means is time to open the hardware vendors brochures and start looking for a replacement on your I/O subsystem. Before you look into making a salesman happy, there is one more thing you have to verify: is your CPU using some sort of clock frequency adjustment technologies, like CPU Stepping or AMD&#8217;s &#8216;Cool&#8217;n&#8217;Quiet&#8217;? These technologies affect the way SQL Server measures time passed and may result in erroneous measurements. The problem is described in KB 931279: <a href="http://support.microsoft.com/kb/931279" target="_blank">http://support.microsoft.com/kb/931279</a>. I have seen the effects of this behavior on some systems and I can say that the results of SQL Server I/O time reports are way, way off chart. When you look in sys.dm\_io\_virtual\_file\_stats and see average I/O write stall times of 245000 milliseconds on a system that works fine, you know it something must be wrong. The cause is that CPU adjust the clock timers during I/O requests and then SQL Server uses wrong clock frequencies to compute the duration of I/O.

The workaround is fairly simple, force the CPU to run a maximum frequency always. First, make sure you are using an &#8216;always on&#8217; power scheme:  
1. Click Start, click Run, type Powercfg.cpl, and then click OK.  
2. In the Power Options Properties dialog box, click Always On in the Power schemes list.</br>  
3. Click OK.</br>

In addition, make sure no third party tools are enabling the CPU clock changes even when the power scheme is set to &#8216;always on&#8217;.