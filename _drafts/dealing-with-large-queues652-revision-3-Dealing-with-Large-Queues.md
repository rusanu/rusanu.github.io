---
id: 655
title: Dealing with Large Queues
date: 2010-03-02T16:35:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/02/652-revision-3/
permalink: /2010/03/02/652-revision-3/
---
On a project I&#8217;m currently involved with we have to handle a constant influx of audit messages for processing. The messages come from about 50 SQL Express instances located in data centers around the globe, delivered via Service Broker into a processing queue hosted on a mirorred database where an activated procedure shreds the message payload into relational tables. These tables are in turn replicated with transactional replication into a data warehouse database. after that the messages are deleted from the processing servers, as replication is set up not to replicate deletes. The system must handle a constant average rate of about 200 messages per second, 24&#215;7, with spikes going up to 2000-3000 messages per second over periods of minutes to an hour.

When dealing with these relatively high volumes, it is inevitable that queues will grow during the 2000 msgs/sec spikes and drain back to empty when the incomming rate stabilizez again at the normal 200 msgs/sec rate. As a note, these spikes occur mostly due to connectivity problems that occur wehn the SQL Express hosting machines undergo some adminsitrative change and something is misconfigured, usually an IPSec policy&#8230; Service Broker does an excelent job at handling these non-conectivity periods, retains