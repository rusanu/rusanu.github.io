---
id: 226
title: Replacing Service Certificates that are near expiration
date: 2008-11-26T14:53:04+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/11/26/221-revision-5/
permalink: /2008/11/26/221-revision-5/
---
Service Broker services use certificates for authenticating messages origin and for encrypting messages. I have explained in detail how this authentication works in my earlier post <a href="/2008/11/04/conversations-authentication/" target="_blank">Conversation Authentication</a>. The certificates used for service authentication are most times self-signed certificates created directly by SQL Server using <tt>CREATE CERTIFICATE</tt> and expire by default one year after creation. When these certificates expire they have to be replaced and this article is going to show how to do this replacement with no impact on production systems.

## Identifying the certificates use by services

Here is a recap of the criteria Service Broker uses to pick a certificate to represent an service identity:

  * Service owner and certificate owner must be the same database principal
  * Certificate private key must be encrypted with the database master key
  * Certificate start date and expiry dates must be valid
  * Certificate must be ACTIVE FOR BEGIN_DIALOG = ON

If there are more certificates that satisfy the above criteria then Service Broker will pick the one with the latest expiration date. The following query will show the certificates that can be used by Service Broker and the services they can be used for:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.services</span><span style="color:Black"> s
	</span><span style="color:Gray">join</span><span style="color:Black"> </span><span style="color:Green">sys.certificates</span><span style="color:Black"> c </span><span style="color:Blue">on</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">principal_id </span><span style="color:Gray">=</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">principal_id
	</span><span style="color:Blue">where</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">pvt_key_encryption_type </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'MK'
</span><span style="color:Black">		</span><span style="color:Gray">and</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">is_active_for_begin_dialog </span><span style="color:Gray">=</span><span style="color:Black"> 1
		</span><span style="color:Gray">and</span><span style="color:Black"> </span><span style="color:Fuchsia">GETUTCDATE</span><span style="color:Gray">()</span><span style="color:Black"> </span><span style="color:Gray">BETWEEN</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date </span><span style="color:Gray">AND</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date
		</span><span style="color:Gray">and</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">service_id </span><span style="color:Gray">&gt;</span><span style="color:Black"> 2</span><span style="color:Gray">;</span></pre>

The query filters out the QueryNotifications and EventNotifications services because these two services do not pick certificates base don the service owner but instead they pick the ones used by the owner of the notification subscription.

<!--more-->

You can run this query in any database that has services establishing remote dialogs and quickly see if the certificates used are about to expire.

## Replacing Certificates

When I explained how to replace the <a href="/2008/10/25/replacing-endpoint-certificates-that-are-near-expiration/" target="_blank">certificates for endpoints</a> I showed that the replacement can go smooth and with no downtime for production if it the replacement process takes care of first deploying the certificate to the peers and only after that starts using the new certificate. Same goes for certificates used by services. The replacement process goes as follows:

  1. Create a new certificate to replace the old one but mark the certificate as <tt>ACTIVE FOR BEGIN_DIALOG=OFF</tt>.
  2. Deploy the certificate to the other databases that establish dialogs with the service(s) that are having the certificate replaced.
  3. Activate the new certificate by turning <tt>ACTIVE FOR BEGIN_DIALOG=ON</tt>
  4. Drop the old certificate