---
id: 1740
title: Registry bloat after SQL Server 2012 SP1 installation
date: 2013-02-15T04:57:29+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/02/15/1728-revision-11/
permalink: /2013/02/15/1728-revision-11/
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

But the this problem has a much sinister side effect: it causes growth of the HKLM\Software registry hive. Except for the System hive, all the other registry hives are still restricted in size to a max of 2GB, see [Registry Storage Space](http://msdn.microsoft.com/en-us/library/windows/desktop/ms724881%28v=vs.85%29.aspx):  


> Views of the registry files are mapped in paged pool memory&#8230;The maximum size of a registry hive is 2 GB, except for the system hive.

As the runaway msiexec process bloats the Software registry hive your system may approach the maximum hive size and, even before that maximum is reached, the large hive will consume more and more of the very precious kernel paged pool memory. The system may start to exhibit erratic behavior, complaining about low &#8216;resources&#8217;, closing connections and other symptoms. For an example of how this erratic bahior may manifest, read [Why the registry size can cause problems with your SQL 2012 AlwaysOn/Failover Cluster setup](http://blogs.msdn.com/b/sqljourney/archive/2012/10/25/why-the-registry-size-can-cause-problems-with-your-sql-2012-alwayson-setup.aspx).

<p class="callout float-right">
  If you installed SQL Server 2012 SP1 recently follow the steps in KB2793634 for a fix
</p>

The biggest problem with the registry bloat erratic behavior is that it occurs long after the SQL Server 2012 SP1 installation and is quite difficult, even for expert users, to trace back the causality of the server erratic behavior to the SP1 installation. If you are uncertain if this issue is affecting you, run <tt>dir %SystemRoot%\system32\config</tt> and check the size of the <tt>SOFTWARE</tt> file, which is the storage of this registry hive. If you are indeed dealing with a bloated registry hive, visit [KB2498915: How to Compress &#8220;Bloated&#8221; Registry Hives](http://support.microsoft.com/kb/2498915):

> 1) Boot from a [WinPE disk](http://technet.microsoft.com/en-us/library/cc766093(WS.10).aspx).  
> 2) Open regedit while booted in WinPe, load the bloated hive under HLKM. (e.g. HKLM\Bloated)  
> 3) Once the bloated hive has been loaded, export the loaded hive as a &#8220;Registry Hive&#8221; file with a unique name. (e.g. %windir%\system32\config\compressedhive)  
> a) You can use dir from a command line to verify the old and new sizes of the registry hives.  
> 4) Unload the bloated hive from regedit. (If you get an error here, close the registry editor. Then reopen the registry editor and try again.)  
> 5) Rename the hives so that you will boot with the compressed hive.  
> e.g.  
> <tt>c:\windows\system32\config\ren software software.old</tt>  
> <tt>c:\windows\system32\config\ren compressedhive software</tt>