---
id: 222
title: Replacing Service Certificates that are near expiration
date: 2008-11-26T14:17:53+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/11/26/221-revision/
permalink: /2008/11/26/221-revision/
---
Service Broker services use certificates for authenticating messages origin and for encrypting messages. I have explained in detail how this authentication works in my earlier post <a href="/2008/11/04/conversations-authentication/" target="_blank">Conversation Authentication</a>. The certificates used for service authentication are most times self-signed certificates created directly by SQL Server using <tt>CREATE CERTIFICATE</tt> and expire by default one year after creation. When these certificates expire they have to be replaced and this article is going to show how to do this replacement with no impact on production systems.

<!--more-->

## < </p>