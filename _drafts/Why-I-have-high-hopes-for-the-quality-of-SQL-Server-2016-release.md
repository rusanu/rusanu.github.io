---
id: 2470
title: Why I have high hopes for the quality of SQL Server 2016 release
date: 2016-04-01T04:56:56+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=2470
permalink: /?p=2470
categories:
  - SQL2016
---
SQL Server 2016 will be the first [Cloud First](https://news.microsoft.com/2014/03/27/satya-nadella-mobile-first-cloud-first-press-briefing) release. This is not just a buzz-word fest. I&#8217;ve been through many SQL Server production cycles, and the 2016 one _is_ different. Beyond all marketing smoke screen there are fundamental changes in SQL Server 2016 code, changes in the way the dev team operates, changes in the way features are designed.

The Azure SQL DB v12 was a radical shift in the way the Azure SQL DB product was developed and operated. Prior to this the Azure SQL code base was a derivative of the core SQL Engine code and was based on a rowset level replication technology, specific to Azure SQL DB. This is why you had restriction in Azure SQL DB that all tables must have a clustered index (heaps were not supported). With Azure SQL DB v12 the service has changed to a new technology based on physical database replication (you may know this technology as &#8216;mirroring&#8217;, or &#8216;HADRON&#8217;, or &#8216;AlwaysOn&#8217;, or &#8216;Always ON&#8217;). The Azure SQL DB service became based on the same code as the standalone product. We&#8217;re not talking even branching. The impact was immediate and profound.

<!--more-->

With Azure SQL DB v12 the _development_ branch of the SQL Engine core is snapped periodically and deployed as the Azure SQL DB service. As developers were working on new features targeting SQL Server 2016 release, any check-in they made was actually alive in _production_ Azure SQL DB service in about 2 months after the check in. And if this check-in would create any problem in production, the developer was on the hook for mitigating the issue. Azure SQL DB has no &#8216;ops&#8217; team. The dev team _owns_ the service. Prior SQL Server development cycles favored &#8216;feature&#8217; branches, a code branching strategy in which new features were developed in separate branches and reverse integrated into &#8216;main&#8217; branch when the feature was complete. This strategy would &#8216;shield&#8217; the main branch from the effect of incomplete, partial developed features. The draw back was that features all &#8216;came together&#8217; at the end of the development cycle and inevitable issues would arise. I&#8217;m not talking merge conflict, I&#8217;m talking fundamental incompatibilities of how newly developed feature X would work together with feature Y. With SQL Server 2016 this model was abandoned in favor of everybody working together on the same &#8216;main&#8217; branch. This mean that every single check-in for a partial, unfinished feature had to keep the code quality at the service production code quality. Furthermore, as the code was being checked in, it had to be testable by the development team, but not visible in the live production service (which, remember, runs the very same code). Configuration &#8216;feature switches&#8217; exists in Azure SQL DB which allow for selectively enabling features in production at the desired granularity (single database, account, cluster, data center, everywhere etc), but at the code level this implies strong discipline from the developers as they had to carefully choose where to place <tt>if</tt> statements in the source code to check for the feature switch value and based on that enable or disable a specific behavior.

So now you see why I&#8217;m saying that SQL Server 2016 is truly Cloud First. Every single feature in SQL Server 2016 has been in production in Azure SQL DB for likely more than a year by the time the product will be available. Perhaps some features were not &#8216;live&#8217; in Azure SQL DB, but make no mistake: even those features that are not _officially_ available today in Azure SQL DB where, very likely, tested and tried out in Azure in private mode.

Another