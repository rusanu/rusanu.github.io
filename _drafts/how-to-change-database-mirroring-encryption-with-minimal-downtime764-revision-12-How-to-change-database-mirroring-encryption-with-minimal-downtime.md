---
id: 776
title: How to change database mirroring encryption with minimal downtime
date: 2010-04-23T10:52:37+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/04/23/764-revision-12/
permalink: /2010/04/23/764-revision-12/
---
SQL Server Database Mirroring uses an encrypted connection to ship the data between the principal and the mirror. By default RC4 encryption is enabled and used. The endpoint can be configured to use AES encryption instead, or no encryption. The overhead of using RC4 encryption is quite small, the overhead of using AES encryption is slightly bigger, but not significant. Under well understood conditions, like inside a secured data center, encryption can be safely turned off for a 5-10% increase in speed in mirroring traffic. Note that even with encryption turned off, the traffic is still cryptographically signed (HMAC). Traffic signing cannot be turned off.

To change the encryption used by an endpoint, one has to run the <a href="http://technet.microsoft.com/en-us/library/ms186332.aspx" target="_blank">ALTER ENDPOINT &#8230; FOR DATABASE_MIRRORING (ENCRYPTION = {DISABLED|SUPPORTED|REQUIRED})</a>. Two endpoints must have compatible encryption settings to be able to communicate. The following table shows the compatibility matrix of encryption settings:

<table style="width:90%">
  <colgroup> <col width="25%" style="background-color:lightgray"/> <col width="25%"/> <col width="25%"/> <col width="25%"/> </colgroup> <tr style="background-color:lightgray">
    <th style="background-color:white" />
    
    <th>
      DISABLED
    </th>
    
    <td>
      SUPPORTED</th> 
      
      <th>
        REQUIRED
      </th></tr> </thead> 
      
      <tr>
        <td>
          DISABLED
        </td>
        
        <td>
          CLEAR
        </td>
        
        <td>
          CLEAR
        </td>
        
        <td>
          &#8211;
        </td>
      </tr>
      
      <tr>
        <td>
          SUPPORTED
        </td>
        
        <td>
          CLEAR
        </td>
        
        <td>
          ENCRYPTRED
        </td>
        
        <td>
          ENCRYPTED
        </td>
      </tr>
      
      <tr>
        <td>
          REQUIRED
        </td>
        
        <td>
          &#8211;
        </td>
        
        <td>
          ENCRYPTRED
        </td>
        
        <td>
          ENCRYPTRED
        </td>
      </tr></table>