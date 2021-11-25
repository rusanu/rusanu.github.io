---
id: 1127
title: 'Scalable SQL with reliable messaging vs. MySpace&#8217;s death spiral'
date: 2011-05-31T15:56:33+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/05/31/1119-revision-8/
permalink: /2011/05/31/1119-revision-8/
---
<a href="http://cacm.acm.org/magazines/2011/6/108663-scalable-sql/fulltext" target="_blank">How do large-scale sites and applications remain SQL-based?</a> is a recent article on <a href="http://cacm.acm.org/about-communications" target="_blank">ACM.ORG</a> that highlights the relational SQL Server based web-scale deployment at MySpace. I have talked before about how MySpace uses Service Broker as a reliable messaging backbone to power the communications between +1000 databases, allowing them to scale-out and partition the user information into individual shards. You can see more details at

  * [MySpace Uses SQL Server Service Broker to Protect Integrity of 1 Petabyte of Data](http://rusanu.com/2009/07/26/myspace-uses-sql-server-service-broker-to-protect-integrity-of-1-petabyte-of-data/)
  * [Developing Large Scale Web Applications and Services](http://mschnlnine.vo.llnwd.net/d1/pdc08/WMV-HQ/BB07.wmv)(video)</a>
  * <a href="http://www.slideshare.net/markginnebaugh/myspace-sql-server-service-broker-oct-2009" target="_blank">MySpace SQL Server Service Broker</a> (slides)

This new article uses the MySpace deployment as a case study to counter balance the claim that large web-scale deployments require the use of NoSQL storage because relational database cannot scale. BTW I know the SQL vs. NoSQL discussion is more subtle, but I won&#8217;t enter into details here. I think a good read on _that_ topic is <a href="http://stu.mp/2010/03/nosql-vs-rdbms-let-the-flames-begin.html" target="_blank">NoSQL vs. RDBMS: Let the flames begin!</a>.

But my post today is actually from a completely different angle. Robert Scoble had a post <a href="http://scobleizer.com/2011/03/24/myspaces-death-spiral-due-to-bets-on-los-angeles-and-microsoft/" target="_blank">MySpace’s death spiral: insiders say it’s due to bets on Los Angeles and Microsoft</a> on which it claims that MySpace insider sources blame their loss to Facebook to, amongst other things, the complexity of this infrastructure:

> Workers inside MySpace tell me that this infrastructure, which they say has “hundreds of hacks to make it scale that no one wants to touch” is hamstringing their ability to really compete.

I do no know if the sources quoted by the article are right or wrong, and I was never personally involved with the MySpace deployment, but since I was so closely involved in building SQL Service Broker and as SSB is one of the critical components used by MySpace infrastructure, I am understandably interested in this discussion. My topic of interest is much narrower than the subject of how did Facebook outmaneuver MySpace, my interest is only _Can Service Broker be used as the basis of a web-scale relational SQL deployment **in the real world**?_