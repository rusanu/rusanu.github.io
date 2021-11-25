---
id: 654
title: Dealing with Large Queues
date: 2010-03-02T16:28:45+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/02/652-revision-2/
permalink: /2010/03/02/652-revision-2/
---
On a project I&#8217;m currently involved with we have to handle a constant influx of audit messages for processing. The messages come from about 50 SQL Express instance, delivered via Service Broker into a queue hosted on a mirorred database where an activated procedure shreds the message payload into rleational tables. These tables are in turn replicated with transactional replication into a data warehouse database. The system must handle a constant average rate of about 200 messages per second, 24&#215;7, with spikes going up to 2000-3000 messages per second over periods of minutes to an hour.