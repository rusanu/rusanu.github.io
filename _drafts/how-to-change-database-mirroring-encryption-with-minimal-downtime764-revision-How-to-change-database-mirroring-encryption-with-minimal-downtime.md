---
id: 765
title: How to change database mirroring encryption with minimal downtime
date: 2010-04-23T09:02:38+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/04/23/764-revision/
permalink: /2010/04/23/764-revision/
---
SQL Server Database Mirroring uses an encrypted connection to ship the data between the principal and the mirror. By default RC4 encryption is enabled and used. The endpoint can be configured to use AES encryption instead, or no encryption. The overhead of using RC4 encryption is quite small, the overhead of using AES encryption is slightly bigger, but not significant. Under well understood conditions, like inside a secured data center, encryption can be safely turned off for a 5-10% increase in speed in mirroring traffic. Note that even with encryption turned off, the traffic is still cryptographically signed (HMAC). Traffic signing cannot be turned off.

To change the encryption used by an endpoint, one has to run the <a href="http://technet.microsoft.com/en-us/library/ms186332.aspx" target="_blank">ALTER ENDPOINT &#8230; FOR DATABASE_MIRRORING (ENCRYPTION = {DISABLED|SUPPORTED|REQUIRED}</a>. Two endpoints must have compatible encryption settings to be able to communicate:

<table>
  <
</table>