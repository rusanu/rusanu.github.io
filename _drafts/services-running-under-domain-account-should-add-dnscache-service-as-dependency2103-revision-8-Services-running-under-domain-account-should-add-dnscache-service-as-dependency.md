---
id: 2112
title: Services running under domain account should add dnscache service as dependency
date: 2013-08-27T03:43:04+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/08/27/2103-revision-8/
permalink: /2013/08/27/2103-revision-8/
---
Recently I had to investigate a deadlocked Windows Server 2012 machine. Any attempt to start or stop a service on this machine would freeze in an infinite wait. I did not know about the &#8220;Analyze Wait Chain&#8221; feature in the Task Manager (new since Windows 7), but turns out is quite a life saver. This feature uses the <a href="http://blogs.msdn.com/b/matt_pietrek/archive/2009/04/17/analyze-wait-chain-why-is-my-program-stuck.aspx" target="_blank">Wait Chain Traversal</a> debugging API and is, on the record, <a href="http://blogs.msdn.com/b/matt_pietrek/archive/2009/04/17/analyze-wait-chain-why-is-my-program-stuck.aspx" target="_blank">Matt Pietrek&#8217;s favourite Windows 7 feature</a>. Simply using the Task Manager I was able to see than many programs were waiting on the &#8220;Services and Controller app&#8221; service, which is the SCM (the Service Control Manager):

[<img src="http://rusanu.com/wp-content/uploads/2013/08/wait-chain-scm.png" alt="" title="wait-chain-scm" width="426" height="325" class="alignleft size-full wp-image-2104" />](http://rusanu.com/wp-content/uploads/2013/08/wait-chain-scm.png)  
<!--more-->

The <tt>services.exe</tt> with PID 588 is the SCM &#8220;Services and Controller app&#8221; service. All but one blocked threads in the SCM were waiting on the thread 1072 and thread 1072 was waiting on the LSA service. I took a dump of the PID 588 process and these blocked threads have this short stack:

    
    ntdll!ZwWaitForSingleObject
    ntdll!RtlpWaitOnCriticalSection
    ntdll!RtlpEnterCriticalSectionContended
    services!ScStartServiceAndDependencies
    services!RStartServiceW
    rpcrt4!Invoke
    ...
    </pre>
    <p></p>
    
    
    I'm not a Windows insider, but just by making an educated guess on the function names I can tell that this is a thread servicing an RPC request to start a service, and waiting for a Criticial Section. Knowing from the Task Manager wait chain that the blocking thread is thread 1072 (we could use other means to find the current Critical Section owner, but why bother?). Thread id 1072 stack shows that is waiting on LSA to logon a user:
    
    
        
        ...
        rpcrt4!NdrClientCall3
        sspicli!SspirLogonUser
        sspicli!SspipLogonUser
        sspicli!LsaLogonUser
        sspicli!L32pLogonUser
        sspicli!LogonUserExExW
        services!ScLogonService
        services!ScLogonAndStartImage
        services!ScStartService
        services!ScStartMarkedServices
        services!ScStartServiceAndDependencies
        services!RStartServiceW
        rpcrt4!Invoke
        ...
        
    
    
    Taking a dump on the lsass.exe process shows that there is only one thread active:
    
    
        
        ...
        rpcrt4!NdrClientCall2
        sechost!RStartServiceW
        sechost!StartServiceW
        dnsapi!StartDnsServiceOnDemand
        dnsapi!Query_PrivateExW
        dnsapi!Query_Shim
        dnsapi!DnsQuery_UTF8
        netlogon!NetpSrvOpen
        netlogon!NetpDcGetDcNext
        netlogon!NetpDcGetNameSiteIp
        netlogon!NetpDcGetNameIp
        netlogon!NetpDcGetName
        netlogon!DsIGetDcName
        netlogon!DsrGetDcNameEx2
        kerberos!KerbGetKdcBinding
        kerberos!KerbMakeSocketCallEx
        kerberos!KerbMakeSocketCall
        kerberos!KerbGetAuthenticationTicketEx
        kerberos!KerbGetTicketGrantingTicket
        kerberos!LsaApLogonUserEx2
        lsasrv!NegLogonUserEx2Worker
        lsasrv!NegLogonUserEx2
        lsasrv!LsapCallAuthPackageForLogon
        lsasrv!LsapAuApiDispatchLogonUser
        lsasrv!SspiExLogonUser
        sspisrv!SspirLogonUser
        rpcrt4!Invoke
        ...
        
    
    
    So there it is, the LSA is attempting to start the DNS Client service (aka. <tt>dnscache</tt>) in order to respond to a request to logon an user. This call goes back into SCM and blocks on the Critical Section held by thread 1072. This is a circular wait list, hence a deadlock. Our server was stuck in this state for several days before I did this investigation.
    
    
    From analyzing the problem is clear that any service that runs under a domain account can run into this problem during machine startup. The domain account is required because is the fact that is a domain account that triggers the LSA to query the DNS to find the domain controller for the service account domain.
    
    
    I haven't seen this problem before, so is probably quite rare. Several ducks have to align for this to happen. However, if it happens it is difficult to investigate, it does not time out and heal itself, and it mysteriously vanishes after a reboot. My recommendation is to make any service running as a domain account explicitly dependent on the dnscache service. This way the SCM will ensure the dnscache service is started _before_ asking the LSA to create a logon token for the service attempting to start, thus eliminating the possibility of a deadlock. The downside is that stopping the dnscache service will stop your service too, which may be undesirable in some situations.