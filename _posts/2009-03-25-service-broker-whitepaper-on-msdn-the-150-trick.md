---
id: 305
title: 'Service Broker Whitepaper on MSDN: the 150 trick'
date: 2009-03-25T02:24:59+00:00
author: remus
layout: post
guid: /?p=305
permalink: /2009/03/25/service-broker-whitepaper-on-msdn-the-150-trick/
categories:
  - Announcements
tags:
  - msdn
  - service broker
  - white paper
---
A new <a href="http://sqlcat.com" target="_blank">SQL Customer Advisory Team</a> whitepaper was published recently: <a href="http://msdn.microsoft.com/en-us/library/dd576261.aspx" target="_blank">Service Broker: Performance and Scalability Techniques</a> authored by Michael Thomassy.

The whitepaper documents the experience of a test done in Microsoft labs that measured the message throughput attainable between three initiators pushing data to a target. This scenario resembles a high scale ETL case. The test was able to obtain a rate a nearly 18000 messages per second, which is a rate that can satisfy most high load OLTP environments. To obtain this rate Michael and his team had to overcome the high contention around updates of the dialogs system tables. He presents a very interesting trick: create 149 dialogs that remain unused and only use every 150th. This way the updates done on the system tables occur on different pages and the high PAGELATCH contention on the page containing the dialog metadata during SEND is eliminated. A very clever trick indeed. But this is a typical OLTP insert/update trick and that is the very point of the whitetpaper: that typical OLTP techniques can and **should** be applied to Service Broker performance tuning.