---
id: 1140
title: Scale out SQL Server by using Reliable Messaging
date: 2011-06-01T15:05:46+00:00
author: remus
layout: revision
guid: http://rusanu.com/2011/06/01/1119-revision-20/
permalink: /2011/06/01/1119-revision-20/
---
<a href="http://cacm.acm.org/magazines/2011/6/108663-scalable-sql/fulltext" target="_blank">How do large-scale sites and applications remain SQL-based?</a> is a recent article from Michael Rys (<a href="http://twitter.com/#!/SQLServerMike" target="_blank">Twitter</a>) that highlights the relational SQL Server based web-scale deployment at MySpace. I have talked before about how MySpace uses Service Broker as a reliable messaging backbone to power the communications between +1000 databases, allowing them to scale-out and partition the user information into individual shards. Here are some more details about this architecture:

  * [MySpace Uses SQL Server Service Broker to Protect Integrity of 1 Petabyte of Data](http://rusanu.com/2009/07/26/myspace-uses-sql-server-service-broker-to-protect-integrity-of-1-petabyte-of-data/)
  * [Developing Large Scale Web Applications and Services](http://mschnlnine.vo.llnwd.net/d1/pdc08/WMV-HQ/BB07.wmv)(video)</a>
  * <a href="http://www.slideshare.net/markginnebaugh/myspace-sql-server-service-broker-oct-2009" target="_blank">MySpace SQL Server Service Broker</a> (slides)

This new article uses the MySpace deployment as a case study to counter balance the claim that large web-scale deployments require the use of NoSQL storage because relational database cannot scale. BTW I know the SQL vs. NoSQL discussion is more subtle, but I won&#8217;t enter into details here. I think a good read on _that_ topic is <a href="http://stu.mp/2010/03/nosql-vs-rdbms-let-the-flames-begin.html" target="_blank">NoSQL vs. RDBMS: Let the flames begin!</a>.

Why is reliable messaging a key tenet of implementing a data-partitioned scale-out application? Consider a typical example of an modern web-scale application: users connect, get authenticated and view their own profile, but they are also interested in the status updates or wall messages from their network of friends or followers. It is easy to see ho one partitions the data by user, but how do you partition the &#8216;wall&#8217;? User A is in a partition hosted on node 1, while user B is in a partition hosted by node 2, when User A update his status, how does this show up on User B&#8217;s wall?  
One option is to have the application update the wall of User B when User A changes his status. But this prevents scalability, because now writes have to occur on many nodes and the application has to orchestrate all these writes. Think Lady GaGa updating her status, the application has to update the wall of every follower.  
Another option is to have the application read the status from User A when displaying the wall of User B. This also doesn&#8217;t scale, because reads now occur on many nodes. All those Lady GaGa followers refreshing their home page read the one node hosting her status, and the node is soon overwhelmed.

The solution is to replicate the write on User A&#8217;s status onto User B&#8217;s wall. The application only updates the User A status, and the infrastructure propagates the this update to User B&#8217;s wall. But the critical piece is what replication technology can one use? Web-scale deployments have dismissed traditional replication as no scalable:

  * it depends too tightly on physical location and it cannot adapt to rapid changes of topology (nodes being added and removed)
  * its based on schema defined filtering which is difficult to map to the complex application specific data routing conditions (_this_ update goes to node 2, but _that_ update goes to node 3
  * its very sensitive to schema changes making application upgrade roll outs a big challenge</p> 

Robert Scoble had a post <a href="http://scobleizer.com/2011/03/24/myspaces-death-spiral-due-to-bets-on-los-angeles-and-microsoft/" target="_blank">MySpace’s death spiral: insiders say it’s due to bets on Los Angeles and Microsoft</a> on which it claims that MySpace insiders blame their loss to Facebook to, amongst other things, the complexity of this infrastructure:

> Workers inside MySpace tell me that this infrastructure, which they say has “hundreds of hacks to make it scale that no one wants to touch” is hamstringing their ability to really compete.

I do no know if the sources quoted by the article are right or wrong, and I was never personally involved with the MySpace deployment, but since I was so closely involved in building SQL Service Broker and as SSB is one of the critical components used by MySpace infrastructure, I am understandably interested in this discussion.