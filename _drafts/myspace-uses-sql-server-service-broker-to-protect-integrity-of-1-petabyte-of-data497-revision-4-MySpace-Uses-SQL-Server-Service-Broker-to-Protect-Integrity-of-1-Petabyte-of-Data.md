---
id: 501
title: MySpace Uses SQL Server Service Broker to Protect Integrity of 1 Petabyte of Data
date: 2009-07-26T00:42:03+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/07/26/497-revision-4/
permalink: /2009/07/26/497-revision-4/
---
I just found that Microsoft has published <a href="http://www.microsoft.com/casestudies/Case_Study_Detail.aspx?CaseStudyID=4000004532" target="_blank">a white paper about the way MySpace is using Service Broker</a> on their service as the core message delivery system for the _Service Dispatcher_. We&#8217;re talking here 440 SQL Server instances and over 1000 databases. Quote from the whitepaper:

_Service Broker has enabled MySpace to perform foreign key management across its 440 database servers, activating and deactivating accounts for its millions of users, with one-touch asynchronous efficiency. MySpace also uses Service Broker administratively to distribute new stored procedures and other updates across all 440 database servers through the Service Dispatcher infrastructure._ 

That is pretty impressive. I knew about the MySpace SSB adoption since the days when I was with the Service Broker team. You probably all know my mantra I repeat all the time &#8220;don&#8217;t use [fire and forget](http://rusanu.com/2006/04/06/fire-and-forget-good-for-the-military-but-not-for-service-broker-conversations/), is a bad message exchange pattern and there are scenarios when the database may be taken offline&#8221;? Guess how I found out those &#8216;scenarios&#8217;&#8230; Anyway, I&#8217;m really glad that they also made public some performance numbers. Until now I could only quote the 5000 message per second I can push in my own test test environment. Well, looks like MySpace has some beefier hardware:

_Stelzmuller: “When we went to the lab we brought our own workloads to ensure the quality of the testing. We needed to see if Service Broker could handle loads of 4,000 messages per second. Our testing found it could handle more than **18,000 messages a second**.&#8221;_