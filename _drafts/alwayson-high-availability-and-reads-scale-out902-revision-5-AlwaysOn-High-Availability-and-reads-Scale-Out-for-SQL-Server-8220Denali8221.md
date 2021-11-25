---
id: 907
title: 'AlwaysOn: High Availability and reads Scale Out for SQL Server &#8220;Denali&#8221;'
date: 2010-11-10T21:35:44+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/10/902-revision-5/
permalink: /2010/11/10/902-revision-5/
---
The **AlwaysOn** technology in SQL Server &#8220;Denali&#8221;, also known by the project name &#8220;HADR&#8221; and often called Hadron, it is a huge improvement over its predecessor, Database Mirroing. AlwaysOn is also bassed on physical replication of database by shipping over the transaction log, not only similar to Database Mirroring but actually using the DBM technologies to replicate the database. But AlwaysOn brings three more Aces to the table to make an unbeatable play:

Availability Groups
:   Databases with dependencies on one another fail over together, asa group.

Multiple Secondaries
:   AlwaysOn will allow for multiple standby replicas for each availability group.

Readable Secondaries
:   The standby replicas are accessable for rea-only operations.

## Availability Groups

DBM considers each mirrored database as an individual entity. But often application are deployed on several databases contained in an instance and there are tight dependencies between them (eg. cross database queries and procedures). With DBM one had to set up complicated logic to force failover of all related databases if one individual database was occuring a failover event. AlwaysOn introduces Availability Groups that allows explicit declaration of such dependencies. A failover occurs always as an entire group, all databases in the group fail over together become available on the new primary host.

The individual databases inside an Availability Group are still replicated using the Database Mirroring technology

## Multiple Secondaries

Database Mirroring is a private affair between two hosts. A mirroring session can only have on Principal and one Mirror, and they can switch roles. From a High Availability point of view the risk involved in loosing one of the partner was fairly serious. Until the operations were set in place to either add a new partner or bring back the old partner online, the system would run at the risk of loosing availability on anyfurther incident. With AlwaysOn there can be multiple stand-by secondary availability groups, on multiple hosts. This allows for a much safer operations, where multiple failures could be survived without loss of availability. This reduces the cost of achiving &#8216;five nines&#8217; availability numbers, and eases planning, deployment and operating of mission critical systems.

## Readable Secondaries</p>