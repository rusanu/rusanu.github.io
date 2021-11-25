---
id: 786
title: How to change database mirroring encryption with minimal downtime
date: 2010-04-23T11:53:40+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/04/23/764-revision-22/
permalink: /2010/04/23/764-revision-22/
---
SQL Server Database Mirroring uses an encrypted connection to ship the data between the principal and the mirror. By default RC4 encryption is enabled and used. The endpoint can be configured to use AES encryption instead, or no encryption. The overhead of using RC4 encryption is quite small, the overhead of using AES encryption is slightly bigger, but not significant. Under well understood conditions, like inside a secured data center, encryption can be safely turned off for a 5-10% increase in speed in mirroring traffic. Note that even with encryption turned off, the traffic is still cryptographically signed (HMAC). Traffic signing cannot be turned off.

To change the encryption used by an endpoint, one has to run the <a href="http://technet.microsoft.com/en-us/library/ms186332.aspx" target="_blank">ALTER ENDPOINT &#8230; FOR DATABASE_MIRRORING (ENCRYPTION = {DISABLED|SUPPORTED|REQUIRED})</a>. Two endpoints must have compatible encryption settings to be able to communicate. The following table shows the compatibility matrix of encryption settings:

<table style="width:90%; margin-left:auto;margin-right:auto;border: 1px solid #ccc;border-collapse:collapse">
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
          ENCRYPTED
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
          ENCRYPTED
        </td>
        
        <td>
          ENCRYPTED
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
        When you run the ALTER ENDPOINT statement and change the encryption settings the endpoint is going to be stopped and restarted during the ALTER statement. All existing connections will be disconnected. A database mirroring session may immediately re-connect and not react to this short disruption in any fashion visible to the user.
      </p>
      
      <p>
        The safest way to change an existing mirroring session that uses encryption to no longer encrypt the traffic, when there is no witness, would be like this:
      </p>
      
      <ol>
        <li>
          Change the mirror endpoint to SUPPORTED
        </li>
        <li>
          Change the principal endpoint to DISABLED
        </li>
        <li>
          Change the mirror endpoint to DISABLED
        </li>
        <li>
          Verify that the connections are encrypted, check encryption_algorithm column in the <a href="http://technet.microsoft.com/en-us/library/ms189796.aspx" target="_blank">sys.dm_db_mirroring_connections</a> DMV.
        </li>
      </ol>
      
      <p>
        If a the mirroring session involves a witness, then it too must have the endpoint set to a compatible encryption setting:
      </p>
      
      <ol>
        <li>
          Change the witness endpoint to SUPPORTED
        </li>
        <li>
          Change the mirror endpoint to SUPPORTED
        </li>
        <li>
          Change the principal endpoint to DISABLED
        </li>
        <li>
          Change the witness endpoint to DISABLED
        </li>
        <li>
          Change the mirror endpoint to DISABLED
        </li>
        <li>
          Verify that the connections are encrypted, check encryption_algorithm column in the <a href="http://technet.microsoft.com/en-us/library/ms189796.aspx" target="_blank">sys.dm_db_mirroring_connections</a> DMV.
        </li>
      </ol>
      
      <p>
        Note that if automatic failover is enabled then at the moment the principal endpoint is changed, it is possible for automatic failover to occur, given that for a brief moment the mirror and the witness will have a quorum.
      </p>
      
      <h2>
        How to troubleshoot if something goes wrong
      </h2>
      
      <p>
        Attach Profiler to all instances involved in the mirroring session and open a trace that listens for the <a href="http://msdn.microsoft.com/en-us/library/ms190746.aspx" target="_blank">Audit Database Mirroring Login Event Class</a> (on SQL 2005 use the <a href="http://msdn.microsoft.com/en-us/library/ms190261%28v=SQL.100%29.aspx" target="_blank">Audit Broker Login Event Class</a> event instead, it <strong>will</strong> trace the DBM sessions). If you did a mistake during the ALTER ENDPOINT changes and ended up with incompatible settings, there will be an event generated visible in Profiler. The event Text will contain an error message explaining why the endpoints cannot connect.
      </p>