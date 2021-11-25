---
id: 2503
title: A Brief introduction to Containers
date: 2016-07-09T06:03:07+00:00
author: remus
layout: revision
guid: http://rusanu.com/2016/07/09/2490-revision-v1/
permalink: /2016/07/09/2490-revision-v1/
---
Lately I found myself fascinated with Docker and containers. They are now experiencing a popularity surge and one can find all sort of claims being made about what containers are and how they can solve every problem you have with software development, deployment and operations. Microservices, serverless and containerization seem to be _the_ theme of 2016.  


## Namespaces

Linux containers are not new, by any stretch. The foundations have been around in Linux systems since the introduction of namespaces in 2002. Namespaces allow a Linux process to have its own view of several resources: 

  * mount points and filesystem
  * processes
  * network stack
  * inter-process communication
  * hostname
  * user IDs (since Linux 3.8)
  * cgroups (since Linux 4.6)

Namespaces are not a virtualization technology , there is no hypervisor involved. Namespaces provide resource isolation, so that a process in namespace A opens and modifies <tt>/etc/httpd/conf.d</tt> and a process in namespace B opens and modifies the same \*name\* <tt>/etc/httpd/conf.d</tt>, the two processes are actually operating on two distinct, unrelated files. If you look at the list above you&#8217;ll notice that when all those resources are isolated between namespaces, a namespace feels much like an entire OS. If group of processes share a common namespace in which all the binaries, configuration files, data files, network configuration, hostname etc are shared among the processes in the namespace yet isolated from all other namespaces, the namespace feels and acts like a de-facto isolated host.

<p class="callout float-right">
  <b>There is no virtual hardware emulation</b> in a namespace.
</p>

Processes running inside the namespace get all the benefits of isolation and it &#8216;feels&#8217; like they own their own host. But, unlike in a true virtual machine, there is no hardware emulation. System calls are handled by the OS hosting the namespaces, and the OS interacts directly with the hardware layer. Of course the OS itself _can_ (and often is) be running inside a virtual machine so the &#8216;native&#8217; hardware the OS sees is actually virtualized, but that is beside the point.

## Control Groups

**cgroups** is an abbreviation of <tt>control groups</tt>, which is the Linux concept for controlling the resources a group of processes can consume. **cgroups** provide:

  * **limits**: impose maximum on the amount of memory, CPU and IO the group can consume.
  * **prioritization**: allow some groups to have priority on CPU scheduling and IO access.
  * **accounting**: measure the resources consumed by a group.
  * **control**: allows for freezing, checkpointing and restarting entire groups.

On Windows systems [Job Objects](https://msdn.microsoft.com/en-us/library/windows/desktop/ms684161(v=vs.85).aspx) offer similar features to cgroups.

## Containers

With the namespace isolation one can can give a set of processes the illusion of owning their own host, and have the capability to manage them via cgroups. Containers take these basic features and package them in an usable form. This is where things get interesting, because there is actually no concept of a container object in Linux. Instead we have several competing technologies, and containers are an umbrella term for the the ability to leverage namespaces and cgroups in a useful and meaningful manner.