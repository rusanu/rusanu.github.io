---
id: 778
title: How to change database mirroring encryption with minimal downtime
date: 2010-04-23T11:27:09+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/04/23/764-revision-14/
permalink: /2010/04/23/764-revision-14/
---
SQL Server Database Mirroring uses an encrypted connection to ship the data between the principal and the mirror. By default RC4 encryption is enabled and used. The endpoint can be configured to use AES encryption instead, or no encryption. The overhead of using RC4 encryption is quite small, the overhead of using AES encryption is slightly bigger, but not significant. Under well understood conditions, like inside a secured data center, encryption can be safely turned off for a 5-10% increase in speed in mirroring traffic. Note that even with encryption turned off, the traffic is still cryptographically signed (HMAC). Traffic signing cannot be turned off.

To change the encryption used by an endpoint, one has to run the <a href="http://technet.microsoft.com/en-us/library/ms186332.aspx" target="_blank">ALTER ENDPOINT &#8230; FOR DATABASE_MIRRORING (ENCRYPTION = {DISABLED|SUPPORTED|REQUIRED})</a>. Two endpoints must have compatible encryption settings to be able to communicate. The following table shows the compatibility matrix of encryption settings:

<table style="width:90%; margin-left:auto;margin-right:auto;border: 1px solid gray">
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
      
      <p>
        The default setting for an endpoint is ENCRYPTION = REQUIRED, which enforces encryption and refuses to connect to an endpoint that has disabled encryption.
      </p>
      
      <h2>
        Changing encryption settings on an existing endpoint
      </h2>
      
      <p>
        If you have a running mirroring session and want to change the settings to squeeze the extra 5-10% you can expect from removing RC4 encryption, then chances are you deployed the endpoint with the default encryption settings, namely REQUIRED. If you don&#8217;t know the current endpoint settings you can always check the <a href="http://msdn.microsoft.com/en-us/library/ms190278.aspx" target="_blank">sys.database_mirroring_endpoints</a> metadata catalog. When encryption is REQUIRED the encryption_algorithm column is one of 1,2,5 or 6. When encryption is SUPPORTED the encryption_algorithm column is one of 3,4,7 or 8. When is DISABLED the encryption_algorithm is 0 and the is_encryption_Enabled column changes to 0. To force the traffic to be unencrypted at least one endpoint has to have ENCRYPTION = DISABLED.
      </p>
      
      <p>
        When you run the ALTER ENDPOINT statement and change the encryption settings
      </p>