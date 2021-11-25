---
id: 2105
title: Services running under domain account should add dnscache service as dependency
date: 2013-08-27T03:11:15+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/27/2103-revision/
permalink: /2013/08/27/2103-revision/
---
Recently I had to investigate a deadlocked Windows Server 2012 machine. Any attempt to start or stop a service on this machine would freeze in an infinite wait. I did not know about the &#8220;Analyze Wait Chain&#8221; feature in the Task Manager (new since Windows 7), but turns out is quite a life saver. Simply using the Task Manager I was able to see than many programs were waiting on the &#8220;Services and Controller app&#8221; service, which is the SCM (the Service Control Manager)