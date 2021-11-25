---
id: 484
title: Fix slow application startup due to code sign validation
date: 2009-07-24T10:58:28+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/07/24/483-revision/
permalink: /2009/07/24/483-revision/
---
Sometimes you are faced with applications that seem to take ages to start up. Usually they freeze for about 30-40 seconds and then all of the sudden they come to live. This happens for both native and managed application and it sometimes manifest as an IIS/ASP/ASP.Net AppPool starting up slow on the first request. The very first thing I always suspect is code signing verification. When a signed module is checked the certification verification engine may consider that the Certificate Revocation List (CRL) it posses is obsolete and attempt to download a new one. For this it connects to the internet. The problem occurs when the connectivity is either slow, or blocked for some reason. By default the verification engine will time out after 15 seconds and resume with the old, obsolete, CRL it has. The timeout can occur several times, adding up to start up times of even minutes. This occurs completely outside of the control of the application being started, its modules are not even properly wired up in memory so there is no question of application code yet running.

The information on this subject is scarce to say the least. Luckily there is an TechNet article that describes not only the process occuring, but also the controlling parameters: <a href="http://technet.microsoft.com/en-us/library/bb457027.aspx" target="_blank">Certificate Revocation and Status Checking</a>. To fix the problem on computers with not conectivity, registry settings have to be modified in the <tt>HKLMSOFTWAREMicrosoftCryptographyOID<br /> EncodingType 0CertDllCreateCertificateChainEngineConfig</tt> key:

ChainUrlRetrievalTimeoutMilliseconds
:   This is the individual CRL check HTTP call timeout. If is 0 or not present the default value of 15000 seconds is used. Change this timeout to a reasonable value like 200 milliseconds.

ChainRevAccumulativeUrlRetrievalTimeoutMilliseconds
:   This is the aggregate CRL retrieval timeout. If set to 0 or not present the default value of 20 seconds is used. Change this timeout to a value like 500 milliseconds.

With these two changes the code signing verification engine will timeout the CRL refresh operation in 500 milliseconds. If the connectivity to the certificate authority site is bad, this will dramatically increas