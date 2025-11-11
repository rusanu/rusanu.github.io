---
id: 902
title: 'AlwaysOn: High-Availability and reads Scale-Out'
date: 2010-11-11T10:48:41+00:00
author: remus
layout: post
guid: /?p=902
permalink: /2010/11/11/alwayson-high-availability-and-reads-scale-out/
categories:
  - Announcements
  - SQL 2012
---
Along with SQLPASS Summit 2010 announcements on SQL Server &#8220;Denali&#8221; features the MSDN site has published preliminary content on some of these features. Going over the <a href="http://msdn.microsoft.com/en-us/library/ff877884%28v=SQL.110%29.aspx" target="_blank">&#8220;HADR&#8221; Overview (SQL Server)</a> content we can get an early idea about this feature. This post summarizes the AlwaysOn technology, and compares it with its predecessor and close cousin, Database Mirroring. For brevity, I am intentionally omitting a lot of details.

The **AlwaysOn** technology in SQL Server &#8220;Denali&#8221;, also known by the project name &#8220;HADR&#8221; and often called Hadron, it is a huge improvement over its predecessor, Database Mirroring. Like Mirroring, AlwaysOn is also based on physical replication of database by shipping over the transaction log. In fact, it is not only similar to Database Mirroring but actually using the DBM technologies to replicate the database. The steps to set up AlwaysOn contain the steps to set up a Mirroring sessions, and the mirroring endpoints, catalog views and DMVs are still used to set up and monitor AlwaysOn. But AlwaysOn brings three more Aces to the table to make an unbeatable play:<!--more-->

Availability Groups
:   Databases with dependencies on one another fail over together, as a group.

Multiple Secondaries
:   AlwaysOn will allow for multiple standby replicas for each availability group.

Readable Secondaries
:   The standby replicas are accessible for read-only operations.

The following table shows the main differences between AlwaysOn and Database Mirroring:

<table id="_dbmcmp">
  <tr>
    <th>
      Feature
    </th>
    
    <th>
      Mirrroring
    </th>
    
    <th>
      AlwaysOn
    </th>
  </tr>
  
  <tr>
    <td>
      Unit of failover
    </td>
    
    <td>
      Single Database
    </td>
    
    <td>
      Availability Group
    </td>
  </tr>
  
  <tr>
    <td>
      Secondary Access
    </td>
    
    <td>
      Database Snapshots
    </td>
    
    <td>
      Read-Only access
    </td>
  </tr>
  
  <tr>
    <td>
      Quorum and failover
    </td>
    
    <td>
      Witness
    </td>
    
    <td>
      Windows Server Clustering
    </td>
  </tr>
  
  <tr>
    <td>
      Number of secondaries
    </td>
    
    <td>
      Exactly one
    </td>
    
    <td>
      Number of nodes in the cluster
    </td>
  </tr>
</table>

## Availability Groups

DBM considers each mirrored database as an individual entity:

[<img class="aligncenter size-full wp-image-927" title="Database Mirroring Setup" src="/wp-content/uploads/2010/11/dbm_setup.png" alt="" width="90%" />](/wp-content/uploads/2010/11/dbm_setup.png)

But often application are deployed on several databases contained in an instance and there are tight dependencies between them (eg. cross database queries and procedures). With DBM one had to set up complicated logic to force failover of all related databases if one individual database was occurring a failover event. AlwaysOn introduces Availability Groups that allows explicit declaration of such dependencies. A failover occurs always as an entire group, all databases in the group fail over together become available on the new primary host.

The individual databases inside an Availability Group are still replicated using the Database Mirroring technology:

[<img class="aligncenter size-full wp-image-930" title="AlwaysOn Setup" src="/wp-content/uploads/2010/11/hadr_setup.png" alt="" width="90%" />](/wp-content/uploads/2010/11/hadr_setup.png)

## Multiple Secondaries

Database Mirroring is a private affair between two hosts. A mirroring session can only have on Principal and one Mirror, and they can switch roles. From a High Availability point of view the risk involved in losing one of the partner was fairly serious. Until the operations were set in place to either add a new partner or bring back the old partner online, the system would run at the risk of loosing availability on any further incident. With AlwaysOn there can be multiple stand-by secondary availability groups, on multiple hosts. This allows for a much safer operations, where multiple failures could be survived without loss of availability. This reduces the cost of achieving &#8216;five nines&#8217; availability numbers, and eases planning, deployment and operating of mission critical systems.

The figure bellow show a possible AlwaysOn deployment on a 4 node cluster:

  * Availability Group 1 contains 2 databases and 3 nodes of the cluster have joined this availability group. The SQL Server instance on node A is the the primary replica and the ones nodes B and C each host an availability replica.
  * Availability Group 2 contains one database, it has the primary availability group running on the SQL Server instance on node B of the cluster and an availability secondary replica on the instance on node A of the cluster.
  * Availability Group 3 contains five databases, it has the primary availability group running on the SQL Server instance on node D of the cluster and an availability secondary replica on the instance on node C of the cluster.
  * Availability Group 4 contains two databases, it has the primary availability group running on the SQL Server instance on node B of the cluster and an availability secondary replica on the instance on node D of the cluster.

[<img class="aligncenter size-full wp-image-935" title="Cluster Configuration" src="/wp-content/uploads/2010/11/cluster_setup.png" alt="" width="90%" />](/wp-content/uploads/2010/11/cluster_setup.png)

## Readable Secondaries

Secondary replicas are readable in real-time. All read-only operations are allowed on databases in the secondary availability groups. Coupled with the capability to have multiple secondaries, this gives a very nice solution for Scale Out reads. Unlike Database Mirroring database snapshot solution, that was giving a point-in-time snapshot view of the database, readable secondaries allow for real query access to the content of the secondary database, in real-time for up-to-date changes. Access is read-only, no updates are permitted, and all queries run automatically under snapshot isolation model (lock hints and explicitly set isolation levels are ignored). The queries always get a transactionally consistent view of the data, as fresh as allowed by the underlying mirroring session that continuously updates the database. <span style="text-decoration: line-through;">With synchronous mirroring this means that the scale-out achieves the all elusive golden standard of reads that are always perfectly consistent with the last committed writes *on any replica*, with no lag! So far this was impossible to achieve with any other scale-out technology</span> [_Correction 11/22: As Gopal points out bellow, this is still not achievable because synchronous mirroring only requires the shipped log to be hardened to disk, not redone. You can get pretty close, but but the application still needs to be prepared to handle stale reads_].

The following image shows a possible AlwaysOn reads Scale-Out deployed on a 3 node cluster to serve a web farm. Read requests can be server by any of the availability replicas, including the Primary Replica. Write requests have to be all routed to the Primary Replica:

[<img class="aligncenter size-full wp-image-937" title="Read Scale Out" src="/wp-content/uploads/2010/11/read_scale_out.png" alt="" width="90%" />](/wp-content/uploads/2010/11/read_scale_out.png)

## What gives

Database Mirroring has always been a viable choice for small businesses. Available in Standard Edition, running on commodity hardware and with minimal licensing cost requirements, it was basically dirt cheap to implement. AlwaysOn is on a different league. Its availability is no longer based on quorum decided by a witness, as in Database Mirroring, but instead is based on <a href="https://www.microsoft.com/windowsserver2008/en/us/failover-clustering-main.aspx" target="_blank">Windows Server Failover Clustering</a>. WSFC is a premium feature available only on Enterprise and DataCenter editions of the Windows Server family, and is supported only on hardware that has passed the <tt>Validate a Configuration Wizard</tt>. Small businesses can still deploy basic Database Mirroring, but they will have to support a significantly higher entry cost to enjoy the advantages of the AlwaysOn technology.