---
id: 67
title: 'Service Broker &#8216;leaked&#8217; target conversation endpoints fix ships in Cumulative Update for SQL Server 2005 SP2'
date: 2007-08-21T01:05:26+00:00
author: remus
layout: post
guid: /2007/08/21/service-broker-leaked-target-conversation-endpoints-fix-ships-in-cumulative-update-for-sql-server-2005-sp2/
permalink: /2007/08/21/service-broker-leaked-target-conversation-endpoints-fix-ships-in-cumulative-update-for-sql-server-2005-sp2/
categories:
  - Announcements
---
<p align="left">
  The CU3 of SQL Server 2005 SP2 was just released on the web, <a href="http://support.microsoft.com/kb/939537" title="http://support.microsoft.com/kb/939537">http://support.microsoft.com/kb/939537</a>
</p>

<p align="left">
  It contains an update to Service Broker:
</p>

<p align="left">
  <table cellSpacing="1" class="table">
    <tr>
      <td>
        50001416
      </td>
      
      <td>
        <a href="http://support.microsoft.com/kb/940260/" class="KBlink">940260</a><span class="pLink"> (http://support.microsoft.com/kb/940260/)</span>
      </td>
      
      <td>
        FIX: Error message when you use Service Broker in SQL Server 2005: &#8220;An error occurred while receiving data: &#8217;64(The specified network name is no longer available.)'&#8221;
      </td>
    </tr>
  </table>
  
  <p align="left">
    &nbsp;
  </p>
  
  <p align="left">
    The title of the fix is derived from the original incident case, but the fix is actualy for the one case that would lead to &#8216;leaked&#8217; conversation endpoints on the target in remote scenarios (communication between two different SQL Server instances) was fixed. The case fixed is when the pattern of message exchange is correct:
  </p>
  
  <p align="left">
    &#8211; Initiator sends one or more messages
  </p>
  
  <p align="left">
    &#8211; Target issue END CONVERSATION and sends the EndDialog message
  </p>
  
  <p align="left">
    &#8211; Initiator receives the EndDialog and responds by issuing it&#8217;s own END CONVERSATION
  </p>
  
  <p align="left">
    The defect was that if the initiator issues the END CONVERSATION within 1 (one) seconds of the target sending the EndDialog message, the target endpoint was &#8216;leaked&#8217;.
  </p>
  
  <p align="left">
    &nbsp;
  </p>
  
  <p align="left">
    Note that this fix does not address local cases (two databases within the same SQL Server instance) nor the case of remote &#8216;incorrect&#8217; message exchange pattern, when the initiator ends the conversation first w/o ever receiving any message from the target.
  </p>