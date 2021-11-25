---
id: 144
title: Replacing Endpoint Certificates that are near expiration
date: 2008-10-25T05:31:38+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/10/25/137-revision-2/
permalink: /2008/10/25/137-revision-2/
---
## Certificates Expiration

In my [previous post](http://rusanu.com/2008/10/23/how-does-certificate-based-authentication-work/) I have explained how Database Mirroring and Service Broker use certificates for endpoint authentication. The only things validated by SSB/DBM on a certificate are the valid-from date and the expiration date. In fact, even if SSB would not validate these dates, the TLS protocol used underneath by SSB/DBM authentication mechanism would validate these dates. In practice the only one that matter is the expiration date since the valid-from date is usually valid from the moment the certificate was created. Although if you follow this blog you know that I have already talked about a problem that may appear with certificates not yet valid, see <a href="http://rusanu.com/2008/08/25/certificate-not-yet-valid" target="_blank">http://rusanu.com/2008/08/25/certificate-not-yet-valid</a>.

If you have in care servers that use Certificate based Authentication for Database Mirroring or Service Broker endpoints you must be prepared to replace the certificates used when they expire. Neither DBM nor SSB will not interrupt an existing connection when the certificate used to authenticate expires, but when a new connection is attempted the expired certificate will prevent it from being established resulting and some difficult to diagnose and troubleshoot errors. Luckily the certificates about to expire ca be identified and replaced before they expire, with minimal impact on a production server.

### Identifying Endpoints that use Certificates

To find out if the Database Mirroring or Service Broker endpoints are using certificates  
based authentication run the following queries:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.database_mirroring_endpoints</span><span style="color:Gray">;
</span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.service_broker_endpoints</span><span style="color:Gray">;
</span>
</pre>

The authentication used by the endpoint is described in the <tt>connection_auth_desc</tt> column. If it says <tt>CERTIFICATE</tt> then the endpoint is using certificates for authentication and you must next check the expiration date of the certificate used. To find out just which one is this certificate, check the <tt>certificate_id</tt> column and then check the <tt>sys.certificates</tt> metadata catalog to see if the certificate used by the endpoint is about to expire. The <tt>expiry_date</tt> column will show the certificate expiration date. If the expiration date is near then you need to prepare the certificate replacement.

### Replacing Certificates used by Endpoints

The replacement of certificates that are near expiration can be a very smooth operation with no impact on production if done correctly. The procedure is identical for Database Mirroring and for Service Broker endpoints:

  1. Create a new certificate.
  2. Backup the newly created certificate.
  3. Copy the certificate to all the peer servers.
  4. Restore the certificate on each peer server in master database, under the same ownership as the old certificate.
  5. Alter the endpoint to use the new certificate.
  6. Drop the old certificate on each peer.
  7. Drop the old certificate on the server hosting the endpoint being updated.

Because the certificates are deployed (exchanged) prior to changing endpoint&#8217;s certificate, this procedure ensures that the communication is only stopped for a very short brief during the ALTER statement at step 5. And since we&#8217;re deploying the certificates on the peer servers using the same owner as the previously used certificate, there is no need to create a new login and to grant CONNECT permission, the old login is reused and it&#8217;s existing permissions are just fine.

<!--more-->

## Example

I have two SQL Server instances that are exchanging Service Broker messages using certificate based authentication endpoints. The two SQL instances are default instances on the machines <tt>REMUSRX64</tt> and <tt>VSQL2K5EXPRESS</tt>. I&#8217;m going to verify if any of the certificates are near expiration and replace any certificate about to expire.

First thing I need to check the endpoints authentication type and find the certificate used, so I&#8217;m runnig the following query on <tt>REMUSX64</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> connection_auth_desc</span><span style="color:Gray">,</span><span style="color:Black"> certificate_id</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.service_broker_endpoints</span></pre>

<div class="post-image">
  <a href="http://test.rusanu.com/wp-content/uploads/2008/10/service_broker_endpoints.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/10/service_broker_endpoints.png" alt="select connection_auth_desc, certificate_id, * from sys.service_broker_endpoints;" title="Click on the image for a full size view" /></a>
</div>

The endpoint metadata shows that is using <tt>CERTIFICATE</tt> authentication and is using the certificate with id 261. So next I can check the certificate expiration date:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Blue">expiry_date</span><span style="color:Gray">,</span><span style="color:Black"> thumbprint</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> master</span><span style="color:Gray">.</span><span style="color:Green">sys.certificates</span><span style="color:Black"> </span><span style="color:Blue">where</span><span style="color:Black"> certificate_id </span><span style="color:Gray">=</span><span style="color:Black"> 261</span><span style="color:Gray">;</span></pre>

<div class="post-image">
  <a href="http://test.rusanu.com/wp-content/uploads/2008/10/certificate_thumbprint.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/10/certificate_thumbprint.png" alt="select expiry_date, thumbprint, * from master.sys.certificates where certificate_id = 261;" title="Click on the image for a full size view" /></a>
</div>

The certificate used by the Service Broker endpoint on <tt>REMUSRX64</tt> is going to expire on November 1st. That is quite near so I better go ahead and replace this certificate. BTW you notice that I have also explicitly selected the certificate thumbprint, more on this later. First I&#8217;m going to create a new certificate that later will be used be the Service Broker endpoint on <tt>REMUSX64</tt> for authentication:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> [REMUSRX64_Nov_2008]
	</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">subject</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'REMUSRX64 Endpoint Identity'</span><span style="color:Gray">,
</span><span style="color:Black">	</span><span style="color:Blue">start_date</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'10/25/2008'</span><span style="color:Gray">;</span></pre>

As you see I did specify a <tt>start_date</tt> in order to avoid the problem I described in my <a href="http://rusanu.com/2008/08/25/certificate-not-yet-valid/" target="_blank">earlier blog post</a> with certificate start date in Eastern hemisphere. Of course you should use a <tt>start_date</tt> value that matches the day you are doing the replacement operation. Next I&#8217;m going to copy the newly created certificate to <tt>VSQL2K5EXPRESS</tt>. First thing I&#8217;ll backup the certificate to a <tt>.CER</tt> file:

<pre><span style="color: Black"></span><span style="color:Blue">backup</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> [REMUSRX64_Nov_2008]
	</span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'REMUSRX64_Nov_2008.CER'</span><span style="color:Gray">;</span></pre>

Next I&#8217;m copying over the <tt>REMUSRX64_Nov_2008.CER</tt> file to <tt>VSQL2K5EXPRESS</tt> machine via a normal file copy operation. BTW, because I did not specify an explicit path the file was created in the same folder where<tt>master</tt> database resides. If the two machines cannot access each other shares I would use something like mail the file or copy it via an USB stick. Certificate files are public and there is no need to protect them during copy operation, so I don&#8217;t need to take any precaution because a malicious user can gain no advantage from obtaining any of my public certificate files. Next thing I need to do is to restore the certificate on <tt>VSQL2K5EXPRESS</tt> under the same ownership (authorization) as the old certificate used for authentication. So first I need to find out what that ownership is, ie. I need to find out who is the user _on <tt>VSQL2K5EXPRESS</tt>_ that owns the old certificate used by <tt>REMUSRX64</tt> for authentication. When I looked up the old used certificate expiration date I also selected the thumbprint, and I&#8217;m going to use that thumbprint now to find the user I need on <tt>VSQL2K5EXPRESS</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> principal_id</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> master</span><span style="color:Gray">.</span><span style="color:Green">sys.certificates
</span><span style="color:Black">	</span><span style="color:Blue">where</span><span style="color:Black"> thumbprint </span><span style="color:Gray">=</span><span style="color:Black"> 0x67BB038B404E24BAA95B28F714C38A39323E1643</span><span style="color:Gray">;</span></pre>

<div class="post-image">
  <a href="http://test.rusanu.com/wp-content/uploads/2008/10/certificate_owner.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/10/certificate_owner.png" alt="select principal_id, * from master.sys.certificates;" title="Click on the image for a full size view" /></a>
</div>

The owner of the certificate is the user with the principal id 6. Why did I use the thumbprint to look up the certificate? Because this is the only bulletproof way to identify the right certificate. I cannot use the name, because the certificate on <tt>VSQL2K5EXPRESS</tt> can have any name, does not have to match the name used on <tt>REMUSRX64</tt>. Next I&#8217;m going to find the name of the user with principal id 6:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> master</span><span style="color:Gray">.</span><span style="color:Green">sys.database_principals
</span><span style="color:Black">	</span><span style="color:Blue">where</span><span style="color:Black"> principal_id </span><span style="color:Gray">=</span><span style="color:Black"> 6</span><span style="color:Gray">;</span></pre>

<div class="post-image">
  <a href="http://rusanu.com/wp-content/uploads/2008/10/principal_name.PNG" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/10/principal_name.png" alt="select * from master.sys.database_principals where principal_id = 6;" title="Click on the image for a full size view" /></a>
</div>

The user is named <tt>REMUSRX64_Endpoint</tt> and with this knowledge I can now restore the newly created certificate copied from <tt>REMUSRX64</tt> on <tt>VSQL2K5EXPRESS</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> [REMUSRX64_Endpoint_Certificate_Nov_2008]
	</span><span style="color:Blue">authorization</span><span style="color:Black"> [REMUSRX64_Endpoint]
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'c:tempREMUSRX64_Nov_2008.CER'</span><span style="color:Gray">;</span></pre>

Because I used the same ownership (authorization clause) on <tt>VSQL2K5EXPRESS</tt> as the old certificate used for authentication I do not need to create a login and grant <tt>CONNECT</tt> permission. The new certificate will authenticate the same login as the old one and thus benefit from the existing permissions.

Now I have in place the deployment needed to switch to the new certificate. Note that during all these operations the the old certificate was left in place and communication continued unhindered, using the old certificate for authentication. It is also important to realize that in this example I have shown how to copy the certificate to only one peer (to <tt>VSQL2K5EXPRESS</tt>). <span style="color:Red">If your instance is communicating with multiple peers, then you must first copy the certificate and do all the operations described above to <b>all</b> the peers before proceeding</span>. With this said, now is the moment we can switch to the new certificate by changing the endpoint properties to use the new certificate on <tt>REMUSRX64</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">alter</span><span style="color:Black"> </span><span style="color:Blue">endpoint</span><span style="color:Black"> broker
	</span><span style="color:Blue">for</span><span style="color:Black"> </span><span style="color:Blue">service_broker</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">authentication</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> [REMUSRX64_Nov_2008]</span><span style="color:Gray">);</span></pre>

With this statement I have switched over to use the new certificate. The endpoint was stopped and started during the <tt>ALTER</tt> statement and this interrupted any existing connection, but the communication resumed immediately and authentication used the new certificate. We can now validate that the communication is authenticated using the new certificate by running this query on <tt>VSQL2K5EXPRESS</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> principal_name</span><span style="color:Gray">,</span><span style="color:Black"> peer_certificate_id</span><span style="color:Gray">,</span><span style="color:Black"> authentication_method</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black">
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.dm_broker_connections</span></pre>

<div class="post-image">
  <a href="http://test.rusanu.com/wp-content/uploads/2008/10/connection_principal.png" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/10/connection_principal.png" alt="select principal_name, peer_certificate_id, authentication_method, * from sys.dm_broker_connections" title="Click on the image for a full size view" /></a>
</div>

The peer <tt>REMUSRX64</tt> was authenticated to the principal name <tt>REMUSRX64_Endpoint</tt> based on the certificate with id 260. You can verify that this is the new certificate. If you&#8217;re curious what does the authentication method <tt>Microsoft Unified Security Proto</tt> mean, it is the name of the SChannel SSPI provider which was used for authentication.

With the switch effective the last thing is to clean up the old certificate. It is now safe to <tt>DROP</tt> the old certificate both on <tt>REMUSRX64</tt> and on <tt>VSQL2K5EXPRESS</tt> as this certificate is no longer used for Service Broker endpoint authentication. I&#8217;m not going to show how to do this, is straightforward.

### Database Mirroring

In case of Database Mirroring endpoints using certificates for authentication the procedure described above is almost identical, the only difference is that you are going to have to look at the <tt>sys.database_mirroring_endpoints</tt> metadata catalog instead of <tt>sys.service_broker_endpoints</tt>. Also note that if you have a witness in your deployment, then my earlier remark about deploying to all peers before altering the endpoint to switch to the new certificate applies to you, since the endpoint has at least two peers. Also because Database Mirroring endpoints tend to be created together as part of mirroring session deployment, the certificates used tend to have near expiration dates. This procedure showed how to replace the certificate used by one endpoint and is is very likely that you will need to repeat the procedure on all the instances involved in a mirroring session: principal, mirror, witness.