---
id: 583
title: 'bugcollect.com: better customer support'
date: 2009-10-28T21:57:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/10/28/578-revision-5/
permalink: /2009/10/28/578-revision-5/
---
I am a developer, I write applications for fun and profit, and I&#8217;ve been doing this basically my whole professional life. Over the years I&#8217;ve learned that is important to understand the problems my users face. What are the most common issues, how often do they happen, who encounters them most often. I tried the approach of logging into a text file and then ask my users to send me the log file. I&#8217;ve tried sending mail automatically from my application. It was better than nothing, I guess. But my inbox just doesn&#8217;t scale to hundreds of messages that may happen after a &#8230; stormy release.

This is why I created for myself an online service for application crash reporting. Applications can submit incident reports online, and the service will collect them, aggregate them and do some initial analysis, waiting for me to come online and check the status. I had been using this service in my applications over the past year and I think that if I find it so useful, perhaps you will too. So I&#8217;ve invested more resources into this, made it into a commercial product and put it out for everyone:[http://bucollect.com](bugcollect.com).

After an application is ready and published bugcollect.com offers a private channel for collecting logging and crash reporting information. bugcollect.com analyzes crash reports and aggregates similar problems into buckets, groups incidents reported by the same source, helping the development team to focus on the most frequent crashes and problems. Developers get immediate feedback if a new release has a problem and they don&#8217;t have to ask for more information, like application saved error logs, problem description or incident screen shots. Developers can also set up a response to an incident bucket, this response will be sent by bugcolect.com to any new incident report that falls into the same bucket. The application can then interpret this response and display feedback to the user, eg. it can instruct him about a new download available that fixes the problem.

bugcollect.com reporting differs from system crash reporting like iPhone crash, Mac &#8216;send to apple&#8217; or Windows Dr. Watson because it is application initiated. An application can decide to submit a report anytime it wishes, typically in an exception catch block. All reports submitted to bugcollect.com are private and can be viewed only by the account owner, the application development team.

bugcollect.com features a public RESTful XML based API for submitting reports. There are already available client libraries for .Net and Java, as well as appender components for log4net and log4j. More client libraries are under planning or under development and an iPhone library will be made available soon.