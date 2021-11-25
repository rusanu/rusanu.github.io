---
id: 1729
title: Registry bloat after SQL Server 2012 SP1 installation problems
date: 2013-02-15T04:43:19+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/02/15/1728-revision/
permalink: /2013/02/15/1728-revision/
---
SQL Server 2012 installation has the potential to leave an <tt>msiexec.exe</tt> installer process running after the installation finishes, as described in [Windows Installer starts repeatedly after you install SQL Server 2012 SP1](http://support.microsoft.com/kb/2793634):

> After you install SQL Server 2012 SP1 on a computer, the Windows Installer (Msiexec.exe) process is repeatedly started to repair certain assemblies. Additionally, the following events are logged in the Application log:  
> EventId: 1004  
> Source: MsiInstaller  
> Description: Detection of product &#8216;{A7037EB2-F953-4B12-B843-195F4D988DA1}&#8217;, feature &#8216;SQL\_Tools\_Ans&#8217;, Component &#8216;{0CECE655-2A0F-4593-AF4B-EFC31D622982}&#8217; failed. The resource&#8221;does not exist.
> 
> EventId: 1001  
> Source: MsiInstaller  
> Description: Detection of product &#8216;{A7037EB2-F953-4B12-B843-195F4D988DA1}&#8217;, feature &#8216;SQL\_Tools\_Ans’ failed during request for component &#8216;{6E985C15-8B6D-413D-B456-4F624D9C11C2}&#8217;
> 
> When this issue occurs, you experience high CPU usage.  
> Cause
> 
> This issue occurs because the SQL Server 2012 components reference mismatched assemblies. This behavior causes native image generation to fail repeatedly on certain assemblies. Therefore, a repair operation is initiated on the installer package. 

But the real problem is that this contiguous cycle of start and crash of msiexec has a much sinister side effect: it causes growth of the HKLM\Software registry hive. Except for the System hive, all the other registry hives are still restricted in size to a max of 2GB, see [Registry Storage Space](http://msdn.microsoft.com/en-us/library/windows/desktop/ms724881%28v=vs.85%29.aspx):  


> Views of the registry files are mapped in paged pool memory&#8230;The maximum size of a registry hive is 2 GB, except for the system hive.

As the runaway msiexec process bloats the Software registry hive your system may approach the maximum hive size and, even before that maximum is reached, the large hive will consume more and more of the very precious kernel paged pool memory. The system may start to exhibit erratic behavior, complaining about low &#8216;resources&#8217;, closing connections and other symptoms. For an example of how this erratic bahior may manifest, read [Why the registry size can cause problems with your SQL 2012 AlwaysOn/Failover Cluster setup](http://blogs.msdn.com/b/sqljourney/archive/2012/10/25/why-the-registry-size-can-cause-problems-with-your-sql-2012-alwayson-setup.aspx).