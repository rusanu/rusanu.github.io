---
id: 2405
title: Secure Windows Hadoop Clusters
date: 2015-01-21T05:15:20+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=2405
permalink: /?p=2405
categories:
  - Hadoop
---
With the new 2.6 release Hadoop has the ability to deploy secure clusters on Windows. This allows integration of HDFS and Hadoop security with your existing Windows domains. You can grant permissions to the Hadoop cluster resources and to HDFS files based on groups defined in your Active Directory and you can use transparent single sign on authentication when interacting with a Hadoop cluster or with HDFS.

<!--more-->

# Secure Hadoop Clusters

<p class="callout float-right">
  Read <a href="http://hadoop.apache.org/docs/r2.6.0/hadoop-project-dist/hadoop-common/SecureMode.html">Hadoop in Secure Mode</a> for details about how to configure a Secure Hadoop cluster on Linux.
</p>

One of the requirements for running a secure Hadoop cluster is for the jobs to impersonate the user that submits the job. In YARN this is achieved by the [Secure Container Executors](http://hadoop.apache.org/docs/r2.6.0/hadoop-yarn/hadoop-yarn-site/SecureContainer.html). These containers executors start the containers using the platform specific impersonation mechanism. Under Linux the <tt>LinuxContainerExecutor</tt> uses [setuid](http://en.wikipedia.org/wiki/Setuid) to impersonate the user. On Windows the [S4U](http://msdn.microsoft.com/en-us/magazine/cc188757.aspx) is used.

## Kerberos Authentication for Java on Windows  


## </p> 

## LDAP integration

Secure Hadoop clusters can be integrated with an Active Directory to retrieve user group membership information