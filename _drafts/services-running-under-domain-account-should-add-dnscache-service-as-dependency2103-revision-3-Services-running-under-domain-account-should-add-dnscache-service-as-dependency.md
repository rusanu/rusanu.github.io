---
id: 2107
title: Services running under domain account should add dnscache service as dependency
date: 2013-08-27T03:17:47+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/27/2103-revision-3/
permalink: /2013/08/27/2103-revision-3/
---
Recently I had to investigate a deadlocked Windows Server 2012 machine. Any attempt to start or stop a service on this machine would freeze in an infinite wait. I did not know about the &#8220;Analyze Wait Chain&#8221; feature in the Task Manager (new since Windows 7), but turns out is quite a life saver. Simply using the Task Manager I was able to see than many programs were waiting on the &#8220;Services and Controller app&#8221; service, which is the SCM (the Service Control Manager):

[<img src="http://rusanu.com/wp-content/uploads/2013/08/wait-chain-scm.png" alt="" title="wait-chain-scm" width="426" height="325" class="alignleft size-full wp-image-2104" />](http://rusanu.com/wp-content/uploads/2013/08/wait-chain-scm.png)  
<!--more-->

The <tt>services.exe</tt> with PID 588 is the SCM &#8220;Services and Controller app&#8221; service. All but one blocked threads in the SCM were waiting on the thread 1072 and thread 1072 was waiting on the LSA service. I took a dump of the PID 588 process and these blocked threads have this short stack:

    
    ntdll!ZwWaitForSingleObject
    ntdll!RtlpWaitOnCriticalSection
    ntdll!RtlpEnterCriticalSectionContended
    services!ScStartServiceAndDependencies
    services!RStartServiceW
    rpcrt4!Invoke
    rpcrt4!NdrStubCall2
    rpcrt4!NdrServerCall2
    rpcrt4!DispatchToStubInCNoAvrf
    rpcrt4!RPC_INTERFACE::DispatchToStubWorker
    rpcrt4!RPC_INTERFACE::DispatchToStub
    rpcrt4!LRPC_SBINDING::DispatchToStub
    rpcrt4!LRPC_SCALL::DispatchRequest
    rpcrt4!LRPC_SCALL::QueueOrDispatchCall
    rpcrt4!LRPC_SCALL::HandleRequest
    rpcrt4!LRPC_SASSOCIATION::HandleRequest
    rpcrt4!LRPC_ADDRESS::HandleRequest
    rpcrt4!LRPC_ADDRESS::ProcessIO
    rpcrt4!LrpcServerIoHandler
    rpcrt4!LrpcIoComplete
    ntdll!TppAlpcpExecuteCallback
    ntdll!TppWorkerThread
    kernel32!BaseThreadInitThunk
    ntdll!RtlUserThreadStart
    </pre>
    <p></p>