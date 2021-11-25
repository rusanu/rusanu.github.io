---
id: 233
title: Replacing Service Certificates that are near expiration
date: 2008-11-26T17:26:27+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/11/26/221-revision-9/
permalink: /2008/11/26/221-revision-9/
---
Service Broker services use certificates for authenticating messages origin and for encrypting messages. I have explained in detail how this authentication works in my earlier post <a href="/2008/11/04/conversations-authentication/" target="_blank">Conversation Authentication</a>. The certificates used for service authentication are most times self-signed certificates created directly by SQL Server using <tt>CREATE CERTIFICATE</tt> and expire by default one year after creation. When these certificates expire they have to be replaced and this article is going to show how to do this replacement with no impact on production systems.

## Identifying the certificates use by services

Here is a recap of the criteria Service Broker uses to pick a certificate to represent an service identity:

  * Service owner and certificate owner must be the same database principal
  * Certificate private key must be encrypted with the database master key
  * Certificate start date and expiry dates must be valid
  * Certificate must be ACTIVE FOR BEGIN_DIALOG = </b>ON</b>

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

<ol style="list-style-type:decimal">
  <li>
    Create a new certificate to replace the old one but mark the certificate as <tt>ACTIVE FOR BEGIN_DIALOG=<b>OFF</b></tt>.
  </li>
  <li>
    Deploy the certificate to the other databases that establish dialogs with the service(s) that are having the certificate replaced.
  </li>
  <li>
    Activate the new certificate by turning <tt>ACTIVE FOR BEGIN_DIALOG=<b>ON</b></tt>
  </li>
  <li>
    Drop the old certificate
  </li>
</ol>

## Example

I have two services that are exchanging messages over authenticated and encrypted conversations. The initiator service is named <tt>MyService</tt> and is hosted on the machine <tt>REMUSRX64</tt>. The target service is named <tt>ImageProcessor</tt> and is hosted on the machine <tt>VSQL2K5EXPRESS</tt>. First thing I&#8217;m doing is to check if the certificate used by <tt>MyService</tt> is near expiration:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">thumbprint
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.services</span><span style="color:Black"> s
	</span><span style="color:Gray">join</span><span style="color:Black"> </span><span style="color:Green">sys.certificates</span><span style="color:Black"> c </span><span style="color:Blue">on</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">principal_id </span><span style="color:Gray">=</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">principal_id
	</span><span style="color:Blue">where</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">pvt_key_encryption_type </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'MK'
</span><span style="color:Black">		</span><span style="color:Gray">and</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">is_active_for_begin_dialog </span><span style="color:Gray">=</span><span style="color:Black"> 1
		</span><span style="color:Gray">and</span><span style="color:Black"> </span><span style="color:Fuchsia">GETUTCDATE</span><span style="color:Gray">()</span><span style="color:Black"> </span><span style="color:Gray">BETWEEN</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date </span><span style="color:Gray">AND</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date
		</span><span style="color:Gray">and</span><span style="color:Black"> s</span><span style="color:Gray">.</span><span style="color:Black">service_id </span><span style="color:Gray">&gt;</span><span style="color:Black"> 2</span><span style="color:Gray">;</span></pre>



<div class="post-image">
  <a href="/wp-content/uploads/2008/11/service_cert_expire.png" target="_blank"><img src="/wp-content/uploads/2008/11/service_cert_expire.png" alt="" title="service_cert_expire" width="250" class="alignnone size-thumbnail wp-image-227" /></a>
</div>

The certificate will expire on Dec 12, 2008, which is close enough to warrant replacement now. So I will create a new certificate to replace this one, but I will be careful not to enable it for Service Broker by specifying <tt>ACTIVE FOR BEGIN_DIALOG=<b>OFF</b>:</tt>

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> MyService_2009_certificate
	</span><span style="color:Blue">authorization</span><span style="color:Black"> dbo
	</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">subject</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'My Service Identity'</span><span style="color:Gray">,
</span><span style="color:Black">	</span><span style="color:Blue">start_date</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'2008-11-26'
</span><span style="color:Black">	active </span><span style="color:Blue">for</span><span style="color:Black"> begin_dialog</span><span style="color:Gray">=</span><span style="color:Blue"><b>off</b></span><span style="color:Gray">;
</span></pre>

In my case the service <tt>MyService</tt> is owned by <tt>dbo</tt> so I used the <tt>authorization</tt> clause to specify <tt>dbo></tt> as the newly created certificate owner.

Now that I have the new certificate I can go ahead and deploy it on VSQL2K5EXPRESS. Certificate deployment is done by exporting the certificate into a .cer file, transferring the file to the partner system by an out-of-band method (eg. a file transfer, sent my email, or a ftp operation) and then importing the certificate into the target database:

<pre><span style="color: Black"></span><span style="color:Blue">backup</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> MyService_2009_certificate
	</span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'c:sharedMyService_2009_certificate.cer'</span></pre>

I next copy the file over to the VSQL2K5EXPRESS system:

<div class="post-image">
  <a href="/wp-content/uploads/2008/11/copy_service_new_cert.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/11/copy_service_new_cert.png" alt="" title="copy_service_new_cert" width="250" class="alignnone size-thumbnail wp-image-228" /></a>
</div>

In order to restore the certificate on the VSQL2K5EXPRESS system I need to know database principal used on VSQL2K5EXPRESS to represent the identity of <tt>MyService</tt>. In other words, I need to find out the owner, on VSQL2K5EXPRESS, of the existing certificate still used by <tt>MyService</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Fuchsia">user_name</span><span style="color:Gray">(</span><span style="color:Black">principal_id</span><span style="color:Gray">)</span><span style="color:Black">
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.certificates</span><span style="color:Black">
	</span><span style="color:Blue">where</span><span style="color:Black"> thumbprint </span><span style="color:Gray">=</span><span style="color:Black"> 0x1EF52285C7300D3F2DCB5160E93E2963DAB59D8D</span></pre>

<div class="post-image">
  <a href=/wp-content/uploads/2008/11/cert_find_target_owner.png" target="_blank"><img src=/wp-content/uploads/2008/11/cert_find_target_owner.png" alt="" title="cert_find_target_owner" width="250" class="alignnone size-medium wp-image-229" /></a>
</div>

I can now import the new certificate into VSQL2K5EXPRESS:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> MyService_2009_Certificate
	</span><span style="color:Blue">authorization</span><span style="color:Black"> MyService_user
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'c:tempmyservice_2009_certificate.cer'</span></pre>

Now the new certificate is deployed and ready to use. Because it is owned by the user <tt>MyService_user</tt> on VSQL2K5EXPRESS I do not need to grant SEND permission, the user already has this permission. The only action left to do is to enable the new certificate on REMUSRX64 for <tt>ACTIVE FOR BEGIN_DIALOG=<b>ON</b></tt> so that Service Broker starts using this new certificate:

<pre><span style="color: Black"></span><span style="color:Blue">alter</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> MyService_2009_certificate
	</span><span style="color:Blue">with</span><span style="color:Black"> active </span><span style="color:Blue">for</span><span style="color:Black"> begin_dialog </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Gray">;</span></pre>

This is it. All new conversations initiated by <tt>MyService</tt> after this change are going to use the new certificate for authentication and encryption. Because we already deployed the certificate to the partner service there is no production downtime as the target is prepared to accept the new certificate.

Why isn&#8217;t necessary to mark the new certificate <tt>ACTIVE FOR BEGIN_DIALOG=<b>OFF</b></tt> on VSQL2K5EXPRESS during the deployment? Because REMUSRX64 already has the certificate. So even if VSQL2K5EXPRESS does use the new certificate (_and indeed it will use it to encrypt replies_) it is OK, because REMUSRX64 already has this new certificate and can decrypt the replies.

### Target service

OK, so this is how you replace the certificate used by initiator. How about the target, whow does the procedure change? It doesn&#8217;t. The procedure is absolutely identical, but instead of starting from the initiator service certificate you start from the target. Even though the service authentication model is asymmetric and there are differences between the original deployment in the initiator vs. target role (think REMOTE SERVICE BINDING on initiator vs. SEND permission on target), the steps to replace the certificates used are absolutely identical.

One thing worth mentioning is how to look for target&#8217;s certificates about to expire on initiator:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> r</span><span style="color:Gray">.</span><span style="color:Black">remote_service_name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">name</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date</span><span style="color:Gray">,</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">thumbprint
</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.remote_service_bindings</span><span style="color:Black"> r
	</span><span style="color:Gray">join</span><span style="color:Black"> </span><span style="color:Green">sys.certificates</span><span style="color:Black"> c </span><span style="color:Blue">on</span><span style="color:Black"> r</span><span style="color:Gray">.</span><span style="color:Black">remote_principal_id </span><span style="color:Gray">=</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">principal_id
	</span><span style="color:Blue">where</span><span style="color:Black"> pvt_key_encryption_type </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'NA'
</span><span style="color:Black">		</span><span style="color:Gray">and</span><span style="color:Black"> </span><span style="color:Fuchsia">GETUTCDATE</span><span style="color:Gray">()</span><span style="color:Black"> </span><span style="color:Gray">between</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">start_date </span><span style="color:Gray">and</span><span style="color:Black"> c</span><span style="color:Gray">.</span><span style="color:Black">expiry_date
</span></pre>

This query will show certificates associated with remote service bindings and their start date and expiry date. If you see a certificate near expiration then you should connect to the system hosting the service shown in <tt>remote_service_name</tt> column and start _from there_ the replacement process, just as I&#8217;ve shown. If you are not the administrator of that system then all you can do is contact the administrator of said system and notify her that the certificate is about to expire.

## Anonymous Conversations

Anonymous conversations use only one certificate, that of the target. Since they are intended for cases when the target cannot possibly know all the initiators that are connecting to him, the procedure described above will not work since is impossible to deploy the new certificate to all the partners without knowing them. Since for anonymous conversations the initiator services must have the target service&#8217;s certificate the usual deployment strategy involves the public posting of the certificate used by the target so that any potential initiator can deploy it before initiating the conversation with the target. In practice this usually means either a Web page, a FTP location or an intranet share from where the target service can be downloaded. Since that certificate will expire too, the administrator of the target system must also take necessary precautions to replace it. The procedure in this case will work like this:

<ol style="list-style-type:decimal">
  <li>
    A new certificate is created on the target system.
  </li>
  <li>
    The new certificate is posted publicly with a note specifying that starting from a certain date in the future it will be the new certificate accepted.
  </li>
  <li>
    Initiators have a time period allowed for downloading and deploying the new certificate. They enabling the new certificate immediately (<tt>ACTIVE FOR BEGIN_DIALOG=<b>ON</b></tt>) and dropping the old certificate from their system.
  </li>
  <li>
    At the announced date the old certificate will expire. Any client that had neglected to deploy the new certificate will stop working.
  </li>
</ol>

The target service in the anonymous case needs not to bother with the <tt>ACTIVE FOR BEGIN_DIALOG</tt> clause. This clause controls only when Service Broker is actively picking a certificate to use and in anonymous conversations the target never picks one because the replies are encrypted with the initiator&#8217;s session keys, as I described in more detail in my [Endpoint Authentication](/2008/11/04/conversations-authentication/) article.

Given the fact that in anonymous cases the replacement process is dependent on all the initiator&#8217;s doing deployment of the new certificate, it is best if the replacement is planned and started well ahead of expiration. A good policy would be to create certificates valid for one year and post the new replacement every 6 month.