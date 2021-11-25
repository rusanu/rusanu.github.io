---
id: 130
title: How does Certificate based Authentication work
date: 2008-10-23T06:03:32+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/10/23/129-revision/
permalink: /2008/10/23/129-revision/
---
</h2> 

Service Broker and Database Mirroring may use X.509 certificates for authenticating endpoints as an alternative to NTLM/Kerberos authentication. This alternative is actually the only posible one whenever the servers involved are members of unrelated domains (or aren&#8217;t even members of a domain) and the default Windows based authentication is not possible. For Service Broker this scenario is more of the norm rather the exception, while for Database Mirroring this is the exceptional case. To configure an endpoint to use certificates for authentication you must specify the CERTIFICATE keyword in the CREATE/ALTER ENDPOINT statement:

<pre>CREATE ENDPOINT [mirroring]
	STATE = STARTED
	AS TCP (LISTENER_PORT = 5022)
	FOR DATABASE_MIRRORING (
		AUTHENTICATION = CERTIFICATE [MyCertName],
		ROLE = PARTNER);
</pre>

_&#8216;Certificate based authentication&#8217;_ for Service Broker and Database Mirroring sounds new and esoteric, but is really nothing else just a variation of the SSL protocol used to authenticate web sites. To be strict, SQL Server will use <a href="http://msdn.microsoft.com/en-us/library/aa380516.aspx" target="_blank">TLS</a> not <a href="http://msdn.microsoft.com/en-us/library/aa380124(VS.85).aspx" target="_blank">SSL</a>. In SSL/TLS the certificates are used as a means to authenticate a web site by means of checking a property of the certificate and comparing it with the address of the site. The SSL/TLS protocol in itself does not provide any authentication, it only provides a secure way to transmit a certificate from the server to the client and to establish a common secret later used to encrypt and sign traffic, but it is left to the client to perform the authentication of the visited web site by analyzing the certificate provided by the server. In SSL this authentication is done by checking a certificate property and comparing it with the web site address. If they match, and the certificate is signed by an authority trusted by the client, the client can safely conclude that the server really is the web site it desired to connect to, so in effect the client has authenticated the server.

With SQL Server Service Broker and Database Mirroring authentication things are a bit different. After the TLS protocol has provided the &#8216;client&#8217; with the certificate used by the &#8216;server&#8217; (the one specified in the CREATE/ALTER ENDPOINT statement) the client will search this certificate in the master database. If is found, then client has succesfully authenticated the server: it will use the identity of the certificate owner as the identity of the server. This identity is then authorized by the client by checking the CONNECT permission on the SSB/DBM endpoint. As you can see from my description, even though the protocol used is the same or similar, the actual authentication mechanism used for authentication is very different between the typial SSL web site authetication and SQL Server SSB/DBM one. In SQL Server the authentication is based on the physical deployment of the certificate into the master database of the peer. That is, if the certificate used by the &#8216;server&#8217; is found in the &#8216;client&#8217; master database, then the &#8216;server&#8217; is authenticated.

As you can see, there isn&#8217;t any requirement on the certificate used for authentication by SSB/DBM. Unlike the SSL web site case, there is no property validated to math the &#8216;server&#8217; name, nor is there any check done that the certificate is signed by a trusted authority. I know a number of people I talked with were suprised by the later. But there simply isn&#8217;t a need to verify the certificate chain of signatures agains a trusted authority, because in SSB/DBM case the certificate is deployed upfront and hence it can be trusted because the administrator doing the deployment was trusted. So the only requirement on the certificate is not be expired, and a self-signed certificate offers the same level of protection and security as a certificate signed by a trusted authority. Please note that this applies only to SSB and DBM endpoints and not to normal T-SQL endpoints that are exposed to man-in-the-middle attacks when using self-signed certificates to encrypt the T-SQL traffic. And since we&#8217;re talking about the T-SQL endpoints use of certificates, note that they do **not** use the certificates for authentication, but solely to establish an encrypted channel of communication.

We have mentioned so far several times &#8216;client&#8217; and &#8216;server&#8217; but SSB and DBM do not have these roles, both participants are considered &#8216;peers&#8217; in a SSB/DBM connection. Hence when establishing a connection the authentication is literarly done twice, once in each direction, thus enabling mutual authentication between the two peers that participate in the connection.

### Anonymous authentication

For Service Broker the authentication has to support the scenario when a service is exposed for public access and it allows anybody to connect. Obviously, it would be impossible to physically deploy the certificate used by the peer in such scenario because the list of peers is not known upfront. For such a scenario it is possible to grant CONNECT permission on the Service Broker endpoint to the <tt>public</tt> role. After the TLS protocol has presented to the &#8216;client&#8217; the certificate used by the &#8216;server&#8217;, if the &#8216;client&#8217; cannot find this certificate in master it will check if [public] is granted CONNECT permission on the endpoint. If that is true, then the connection is authorized. To enable this scenario, simply grant CONNECT permission to public:

<pre>GRANT CONNECT ON ENDPOINT::[broker] TO [public];</pre>

For Database Mirroring this scenario is not allowed, since there is no realistic need to establish a mirroring session with a peer that is not known upfront.

## Certificates Validity and Expiration