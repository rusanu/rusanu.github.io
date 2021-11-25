---
id: 195
title: 'Event ID 833: I/O requests taking longer than 15 seconds'
date: 2008-10-28T15:09:17+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=195
permalink: /2008/10/28/event-id-833-io-requests-taking-longer-than-15-seconds/
categories:
  - Troubleshooting
tags:
  - errorlog
  - errors
  - IO
  - knowledge base
  - sql
  - sql server
---
The error 833 is usually associated with hardware or system driver problems and the typical recommendation is to replace the hardware or update the drivers and firmware used. However there is a common scenario that leads to this problem when your hardware is fine and sound.

<!--more-->

The MSDN documents the reason for error 833 as this: &#8220;<tt>This problem can be caused system performance issues, hardware errors, firmware errors, device driver problems, or filter driver intervention in the IO process.</tt>&#8220;. When you encounter this error in your ERRORLOG it means is time to open the hardware vendors brochures and start looking for a replacement on your I/O subsystem. Before you look into making a salesman happy, there is one more thing you have to verify: is your CPU using some sort of clock frequency adjustment technologies, like CPU stepping or Cool&#8217;n&#8217;Quiet? These technologies affect the way SQL Server measures time passed and may result in erroneous measurements. The problem is described in KB 931279: <a href="http://support.microsoft.com/kb/931279" target="_blank">http://support.microsoft.com/kb/931279</a>. I have seen the effects of this behavior on some systems and I can say that the results of SQL Server I/O time reports are way, way off chart. When you look in sys.dm\_io\_virtual\_file\_stats and see average I/O write stall times of 245000 milliseconds on a system that works fine, you know something must be wrong. I guess root cause is clock drift between the CPUs that causes a request submitted on one scheduler to be completed on a different one and the time drift between the schedulers to be added to the I/O duration. A clear indication of this drift causing erroneous I/O results is this message in the ERRORLOG: <tt>The time stamp counter of CPU on scheduler id ... is not synchronized with other CPUs</tt>

The workaround is fairly simple, force the CPU to run a maximum frequency always. First, make sure you are using an &#8216;always on&#8217; power scheme:  
1. Click Start, click Run, type Powercfg.cpl, and then click OK.  
2. In the Power Options Properties dialog box, click Always On in the Power schemes list.  
3. Click OK.</br> 

In addition, make sure no third party tools are enabling the CPU clock changes even when the power scheme is set to &#8216;always on&#8217;.

### Edit Dec. 17 2008

There are a number of CSS blog articles that also mention this problem:

  * <a href="http://blogs.msdn.com/psssql/archive/2008/12/16/how-it-works-sql-server-no-longer-uses-rdtsc-for-timings-in-sql-2008-and-sql-2005-service-pack-3-sp3.aspx" target="_blank">http://blogs.msdn.com/psssql/archive/2008/12/16/how-it-works-sql-server-no-longer-uses-rdtsc-for-timings-in-sql-2008-and-sql-2005-service-pack-3-sp3.aspx</a>
  * <a href="http://blogs.msdn.com/psssql/archive/2006/11/27/sql-server-2005-sp2-will-introduce-new-messages-to-the-error-log-related-to-timing-activities.aspx" target="_blank">http://blogs.msdn.com/psssql/archive/2006/11/27/sql-server-2005-sp2-will-introduce-new-messages-to-the-error-log-related-to-timing-activities.aspx</a>
  * <a href="http://blogs.msdn.com/psssql/archive/2007/08/19/sql-server-2005-rdtsc-truths-and-myths-discussed.aspx" target="_blank">http://blogs.msdn.com/psssql/archive/2007/08/19/sql-server-2005-rdtsc-truths-and-myths-discussed.aspx</a>