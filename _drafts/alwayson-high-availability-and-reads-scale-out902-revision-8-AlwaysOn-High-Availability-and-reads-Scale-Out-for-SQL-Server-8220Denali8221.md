---
id: 911
title: 'AlwaysOn: High Availability and reads Scale Out for SQL Server &#8220;Denali&#8221;'
date: 2010-11-10T22:00:56+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/10/902-revision-8/
permalink: /2010/11/10/902-revision-8/
---
The **AlwaysOn** technology in SQL Server &#8220;Denali&#8221;, also known by the project name &#8220;HADR&#8221; and often called Hadron, it is a huge improvement over its predecessor, Database Mirroing. AlwaysOn is also based on physical replication of database by shipping over the transaction log, not only similar to Database Mirroring but actually using the DBM technologies to replicate the database. But AlwaysOn brings three more Aces to the table to make an unbeatable play:

Availability Groups
:   Databases with dependencies on one another fail over together, asa group.

Multiple Secondaries
:   AlwaysOn will allow for multiple standby replicas for each availability group.

Readable Secondaries
:   The standby replicas are accessible for read-only operations.

## Availability Groups

DBM considers each mirrored database as an individual entity. But often application are deployed on several databases contained in an instance and there are tight dependencies between them (eg. cross database queries and procedures). With DBM one had to set up complicated logic to force failover of all related databases if one individual database was occurring a failover event. AlwaysOn introduces Availability Groups that allows explicit declaration of such dependencies. A failover occurs always as an entire group, all databases in the group fail over together become available on the new primary host.

The individual databases inside an Availability Group are still replicated using the Database Mirroring technology

## Multiple Secondaries

Database Mirroring is a private affair between two hosts. A mirroring session can only have on Principal and one Mirror, and they can switch roles. From a High Availability point of view the risk involved in loosing one of the partner was fairly serious. Until the operations were set in place to either add a new partner or bring back the old partner online, the system would run at the risk of loosing availability on any further incident. With AlwaysOn there can be multiple stand-by secondary availability groups, on multiple hosts. This allows for a much safer operations, where multiple failures could be survived without loss of availability. This reduces the cost of achieving &#8216;five nines&#8217; availability numbers, and eases planning, deployment and operating of mission critical systems.

## Readable Secondaries

Secondary replicas are readable in real-time. All read-only operations are allowed on databases in the secondary availability groups. Coupled with the capability to have multiple secondaries, this gives a very nice solution for Scale Out reads. Unlike Database Mirroring database snapshot solution, that was giving a point-in-time snapshot view of the database, readable secondaries allow for real query access to the content of the secondary database, in real-time for up-to-date changes. Access is read-only, no updates are permitted, and all queries run automatically under snapshot isolation model (lock hints and explicitly set isolation levels are ignored). The queries always get a transactionally consistent view of the data, as fresh as allowed by the underlying mirroring session that continuously updates the database. With synchronous mirroring this means that the scale-out achieves the all elusive golden standard of reads that are always perfectly consistent with the last committed writes \*on any replica\*, with no lag! So far this was impossible to achieve with any other scale-out technology.