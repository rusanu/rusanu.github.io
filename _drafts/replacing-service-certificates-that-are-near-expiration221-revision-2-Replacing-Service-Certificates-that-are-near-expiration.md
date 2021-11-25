---
id: 223
title: Replacing Service Certificates that are near expiration
date: 2008-11-26T14:31:52+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/11/26/221-revision-2/
permalink: /2008/11/26/221-revision-2/
---
Service Broker services use certificates for authenticating messages origin and for encrypting messages. I have explained in detail how this authentication works in my earlier post <a href="/2008/11/04/conversations-authentication/" target="_blank">Conversation Authentication</a>. The certificates used for service authentication are most times self-signed certificates created directly by SQL Server using <tt>CREATE CERTIFICATE</tt> and expire by default one year after creation. When these certificates expire they have to be replaced and this article is going to show how to do this replacement with no impact on production systems.

<!--more-->

## Identifying the certificates use by services

Here is a recap of the criteria Service Broker uses to pick a certificate to represent an service identity:

  * Service owner and certificate owner must be the same database principal
  * Certificate private key must be encrypted with the database master key
  * Certificate start date and expiry dates must be valid
  * Certificate must be ACTIVE FOR BEGIN_DIALOG = ON