---
id: 129
title: How does Certificate based Authentication work
date: 2008-10-23T19:52:33+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=129
permalink: /2008/10/23/how-does-certificate-based-authentication-work/
categories:
  - Tutorials
tags:
  - certificate
  - endoint
  - principal
  - security
  - service broker
  - sql
  - tsql
---
Service Broker and Database Mirroring may use certificates for authenticating endpoints as an alternative to NTLM/Kerberos authentication. This alternative is actually the only possible one whenever the servers involved are members of unrelated domains (or aren&#8217;t even members of a domain) and the default Windows based authentication is not possible. For Service Broker this scenario is more of the norm rather the exception, while for Database Mirroring this is the exceptional case. To configure an endpoint to use certificates for authentication you must specify the CERTIFICATE keyword in the CREATE/ALTER ENDPOINT statement:

<pre><span style="color: Black"></span><span style="color:Blue">CREATE</span><span style="color:Black"> </span><span style="color:Blue">ENDPOINT</span><span style="color:Black"> [mirroring]
	</span><span style="color:Blue">STATE</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">STARTED
</span><span style="color:Black">	</span><span style="color:Blue">AS</span><span style="color:Black"> </span><span style="color:Blue">TCP</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">LISTENER_PORT</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> 5022</span><span style="color:Gray">)
</span><span style="color:Black">	</span><span style="color:Blue">FOR</span><span style="color:Black"> </span><span style="color:Blue">DATABASE_MIRRORING</span><span style="color:Black"> </span><span style="color:Gray">(
</span><span style="color:Black">		</span><span style="color:Blue">AUTHENTICATION</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">CERTIFICATE</span><span style="color:Black"> [MyCertName]</span><span style="color:Gray">,
</span><span style="color:Black">		</span><span style="color:Blue">ROLE</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">PARTNER</span><span style="color:Gray">);</span>
</pre>

_&#8216;Certificate based authentication&#8217;_ for Service Broker and Database Mirroring sounds esoteric, yet is really nothing else but a variation of the SSL protocol used to authenticate web sites. To be strict, SQL Server will use <a href="http://msdn.microsoft.com/en-us/library/aa380516.aspx" target="_blank">TLS</a> not <a href="http://msdn.microsoft.com/en-us/library/aa380124(VS.85).aspx" target="_blank">SSL</a>.

SSL and TLS provide a secure way to transmit a certificate from the server to the client and to establish a common secret later used to encrypt and sign traffic. How this is achieved is perhaps out of the scope of a database development oriented discussion, but if you really want to know the gory details MSDN <a href="http://msdn.microsoft.com/en-us/library/aa374782(VS.85).aspx" target="_blank">documents the process</a> in the SChannel SSPI reference:

<ol style="list-style-type:decimal;">
  <li>
    Client calls <a href="http://msdn.microsoft.com/en-us/library/aa375924(VS.85).aspx" target="_blank">InitializeSecurityContext</a> and sends to the server the output buffer(s).
  </li>
  <li>
    The server calls <a href="http://msdn.microsoft.com/en-us/library/aa374716(VS.85).aspx" target="_blank">AcquireCredentialsHandle</a>. The <i><tt>pAuthData</tt></i> parameter contains an <tt>SCHANNEL_CRED</tt> structure that describes the certificate used by the server for authentication.
  </li>
  <li>
    The server calls <a href="http://msdn.microsoft.com/en-us/library/aa374708(VS.85).aspx" target="_blank">AcceptSecurityContext</a> passing in the buffer(s) provided by the client. Any output buffer is sent back to the client.
  </li>
  <li>
    The client receives the buffer(s) from the server and calls again <a href="http://msdn.microsoft.com/en-us/library/aa375924(VS.85).aspx" target="_blank">InitializeSecurityContext</a> passing in the buffer(s) from the server. If any output buffer results, it is sent to the server.
  </li>
  <li>
    The server receives more buffer(s) from the client and calls again <a href="http://msdn.microsoft.com/en-us/library/aa374708(VS.85).aspx" target="_blank">AcceptSecurityContext</a> passing in the buffer(s) provided by the client. If any out buffer results, it is sent to the client.
  </li>
  <li>
    Steps 4 and 5 are repeated until no more output buffers are produced.
  </li>
  <li>
    The client calls <a href="http://msdn.microsoft.com/en-us/library/aa379340(VS.85).aspx" target="_blank">QueryContextAttributes</a> on the resulted security context and asks for the <tt>SECPKG_ATTR_REMOTE_CERT_CONTEXT</tt> attribute. With this call the client has obtained a copy of the certificate used by the server in step 2 to initiate the authentication process.
  </li>
  <li>
    Further traffic between client and server can be encrypted using the <a href="http://msdn.microsoft.com/en-us/library/aa375390(VS.85).aspx" target="_blank">EncryptMessage</a> and <a href="http://msdn.microsoft.com/en-us/library/aa375348(VS.85).aspx" target="_blank">DecryptMessage</a> functions.
  </li>
</ol>

<!--more-->

The actual content of those &#8216;black box&#8217; buffers exchanged by the client and server during authentication is described in <a href="http://www.ietf.org/rfc/rfc2246.txt" target="_blank">RFC 2246</a>. Or if you prefer a more digestible read go for Eric Rescorla&#8217;s excellent book on the topic <a href="http://www.amazon.com/SSL-TLS-Designing-Building-Systems/dp/0201615983" target="_blank">SSL and TLS</a>. And as a side note, the &#8216;Windows&#8217; authentication is identical but uses the <a href="http://msdn.microsoft.com/en-us/library/aa378748(VS.85).aspx" target="_blank">SPNego</a>, <a href="http://msdn.microsoft.com/en-us/library/aa378749(VS.85).aspx" target="_blank">NTLM</a> or <a href="http://msdn.microsoft.com/en-us/library/aa378747(VS.85).aspx" target="_blank">Kerberos</a> functions instead of the <a href="http://msdn.microsoft.com/en-us/library/aa380123(VS.85).aspx" target="_blank">SChannel</a> ones.

The SSL/TLS protocol in itself does not provide any authentication, it only provides the client with the certificate used by the server and the client is supposed to use this certificate to authenticate the server. In SSL this authentication is done by checking a certificate property and comparing it with the web site address. If they match, and the certificate is signed by an authority trusted by the client, the client can safely conclude that the server really is the web site it desired to connect to, so in effect the client has authenticated the server.

With SQL Server Service Broker and Database Mirroring authentication things are a bit different. After the TLS protocol has provided the &#8216;client&#8217; with the certificate used by the &#8216;server&#8217; (the one specified in the CREATE/ALTER ENDPOINT statement) the client will search this certificate in the master database. If is found, then client has successfully authenticated the server: it will use the identity of the certificate owner as the identity of the peer server. This identity is then authorized by the client by checking the CONNECT permission on the SSB/DBM endpoint. As you can see from my description, even though the protocol used is the same or similar, the actual authentication mechanism used for authentication is very different between the typical SSL web site authentication and SQL Server SSB/DBM one. In SQL Server the authentication is based on the physical deployment of the certificate into the master database of the peer. That is, if the certificate used by the &#8216;server&#8217; is found in the &#8216;client&#8217; master database, then the &#8216;server&#8217; is authenticated.

As you can see, there isn&#8217;t any requirement on the certificate used for authentication by SSB/DBM. Unlike the SSL web site case, there is no property validated to match the &#8216;server&#8217; name, nor is there any check done that the certificate is signed by a trusted authority. I know a number of people I talked with were surprised by the later. But there simply isn&#8217;t a need to verify the certificate&#8217;s chain of signatures against a trusted authority, because in SSB/DBM case the certificate is deployed upfront and hence it can be trusted because the administrator doing the deployment was trusted. So the only requirement on the certificate is not be expired. Also in SSB/DBM a self-signed certificate offers the same level of protection and security as a certificate signed by a trusted authority. Please note that this applies only to SSB and DBM endpoints and not to normal T-SQL endpoints that are exposed to man-in-the-middle attacks when using self-signed certificates to encrypt the T-SQL traffic. And since we&#8217;re talking about the T-SQL endpoints use of certificates, note that they do **not** use the certificates for _client_authentication, but instead they are used to authenticate the _server_ and to establish an encrypted channel of communication. The _client_ is authenticated via name and password (for SQL Logins) or using NTLM/Kerberos (for Windows Logins).

We have mentioned so far several times &#8216;client&#8217; and &#8216;server&#8217; but SSB and DBM do not have these roles, both participants are considered &#8216;peers&#8217; in a SSB/DBM connection. Hence when establishing a connection the authentication is literally done twice, once in each direction, thus enabling mutual authentication between the two peers that participate in the connection.

### Anonymous authentication

For Service Broker the authentication has to support the scenario when a service is exposed for public access and it allows anybody to connect. Obviously, it would be impossible to physically deploy the certificate used by the peer in such scenario because the list of peers is not known upfront. For such a scenario it is possible to grant CONNECT permission on the Service Broker endpoint to the <tt>public</tt> role. After the TLS protocol has presented to the &#8216;client&#8217; the certificate used by the &#8216;server&#8217;, if the &#8216;client&#8217; cannot find this certificate in master it will check if <tt>public</tt> is granted <tt>CONNECT</tt> permission on the endpoint. If that is true, then the connection is authorized. To enable this scenario, simply grant <tt>CONNECT</tt> permission to <tt>public</tt>:

<pre><span style="color: Black"></span><span style="color:Blue">GRANT</span><span style="color:Black"> CONNECT </span><span style="color:Blue">ON</span><span style="color:Black"> </span><span style="color:Blue">ENDPOINT</span><span style="color:Gray">::</span><span style="color:Black">[broker] </span><span style="color:Blue">TO</span><span style="color:Black"> [public]</span><span style="color:Gray">;</span>
</pre>

For Database Mirroring this scenario is not allowed, since there is no realistic need to establish a mirroring session with a peer that is not known upfront.