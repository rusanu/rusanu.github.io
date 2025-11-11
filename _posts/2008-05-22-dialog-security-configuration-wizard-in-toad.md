---
id: 94
title: Dialog Security Configuration Wizard in Toad
date: 2008-05-22T22:22:48+00:00
author: remus
layout: post
guid: /2008/05/22/dialog-security-configuration-wizard-in-toad/
permalink: /2008/05/22/dialog-security-configuration-wizard-in-toad/
categories:
  - Announcements
---
On my <a href="/2008/05/16/viable-tools-for-service-broker/" target="_blank">last post</a> I&#8217;ve said that the Toad tools from Quest lack the option to configure Dialog Security for Service Broker. I stand corrected, as they actualy do. Not only that, but they support both <a href="http://technet.microsoft.com/en-us/library/ms166036.aspx" target="_blank">full security and anonymous security</a>, and the configuration is also done through a wizard. The Dialog Security Configuration Wizard is launched from the context menu of a service.

The wizard is explicit for security and does not combine security and routing, like the <a href="http://ww.codeplex.com/slm" target="_blank">Service Listing Manager</a> does. So to fully configure a pair of services you have to run the Service Broker Application Wizard first and then to follow up with the Dialog Security Configuration Wizard.

The Dialog Security Wizard will guide you through the steps of creating the certificates, set up a database master key, create the users needed, exchange the certificates and set up the remote service binding.

<!--more-->

<div class="post-image">
  <a href='/wp-content/uploads/2008/05/dialogsecurity.png' target="_blank"><img src='/wp-content/uploads/2008/05/dialogsecurity.png' alt="Dialog Security Wizard" width="250" title="Click on the image for a full size view" /></a>
</div>

My one comment on this Wizard is that it somehow contradicts the Broker Aplication Wizard. The Service Broker Application Wizard automatically grants SEND permission to [public] on the target service. In practice this allows any sender to send message to this service. Even if later the Dialog Security Wizard is run, the [public] role is still allowed to SEND on the target service. This is irelevant from a functional point of view, the two services configured through the Wizard will work as expected, but from a security/surface area point of view the service is exposed to a potential attack.