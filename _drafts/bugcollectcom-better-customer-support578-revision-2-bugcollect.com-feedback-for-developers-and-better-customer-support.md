---
id: 580
title: 'bugcollect.com: feedback for developers and better customer support'
date: 2009-10-26T13:18:32+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/10/26/578-revision-2/
permalink: /2009/10/26/578-revision-2/
---
</p> 

After an application is ready and published bugcollect.com offers a private channel for collecting logging and crash reporting information. bugcollect.com analyzes crash reports and aggregates similar problems into buckets, groups incidents reported by the same source, helping the development team to focus on the most frequent crashes and problems. Developers get immediate feedback if a new release has a problem and they don&#8217;t have to ask for more information, like application saved error logs, problem description or incident screen shots. Developers can also set up a response to an incident bucket, this response will be sent by bugcolect.com to any new incident report that falls into the same bucket. The application can then interpret this response and display feedback to the user, eg. it can instruct him about a new download available that fixes the problem.

bugcollect.com reporting differs from system crash reporting like iPhone crash, Mac &#8216;send to apple&#8217; or Windows Dr. Watson because it is application initiated. An application can decide to submit a report anytime it wishes, typically in an exception catch block. All reports submitted to bugcollect.com are private and can be viewed only by the account owner, the application development team.

bugcollect.com features a public RESTful XML based API for submitting reports. There are already available client libraries for .Net and Java, as well as appender components for log4net and log4j. More client libraries are under planning or under development and an iPhone library will be made available soon.