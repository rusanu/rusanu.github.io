---
id: 1121
title: 'Scalable SQL with reliable messaging vs. MySpace&#8217;s death spiral'
date: 2011-05-31T14:27:51+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/05/31/1119-revision-2/
permalink: /2011/05/31/1119-revision-2/
---
<a href="http://cacm.acm.org/magazines/2011/6/108663-scalable-sql/fulltext" target="_blank">How do large-scale sites and applications remain SQL-based?</a> is a recent article on <a href="http://cacm.acm.org/about-communications" target="_blank">ACM.ORG</a> that highlights the relational SQL Server based web-scale deployment at MySpace. I have talked before about how MySpace uses Service Broker as a reliable messaging backbone to power the communications between +1000 databases, allowing them to scale-out and partition the user information into individual shards. You can see more details at

  * [MySpace Uses SQL Server Service Broker to Protect Integrity of 1 Petabyte of Data](http://rusanu.com/2009/07/26/myspace-uses-sql-server-service-broker-to-protect-integrity-of-1-petabyte-of-data/)
  * [Developing Large Scale Web Applications and Services](http://mschnlnine.vo.llnwd.net/d1/pdc08/WMV-HQ/BB07.wmv)(video)</a>
  * <a href="http://www.slideshare.net/markginnebaugh/myspace-sql-server-service-broker-oct-2009" target="_blank">MySpace SQL Server Service Broker</a> (slides)