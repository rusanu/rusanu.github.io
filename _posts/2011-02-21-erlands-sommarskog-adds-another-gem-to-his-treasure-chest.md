---
id: 1073
title: 'Erland&#8217;s Sommarskog adds another gem to his treasure chest'
date: 2011-02-21T22:25:29+00:00
author: remus
layout: post
guid: /?p=1073
permalink: /2011/02/21/erlands-sommarskog-adds-another-gem-to-his-treasure-chest/
categories:
  - Announcements
---
There are probably no articles that I reference more often than Erland&#8217;s. There are questions on forums, discussion groups and [stackoverflow](stackoverflow.com) that come back over and over, again and again. For some of these topics Elands has prepared articles that are, ultimately, **the** article to point anybody for more details on that topic. His coverage is deep, the articles are well structured, and they pretty much exhaust the topic. If somebody still have questions about that topic after reading the article, then I say that person knows too much about SQL Servero &#8230; or not enough. I&#8217;m writing this blog entry not so much as to announce his latest addition <a href="http://www.sommarskog.se/query-plan-mysteries.html" target="_blank">Slow in the Application, Fast in SSMS? Understanding Performance Mysteries</a> (is hard for me to believe anyone that follows my blog does not know about them&#8230;) but more to create for myself a memo pad from where to pick up the link to the articles whenever I need them:

<a href="http://www.sommarskog.se/dynamic_sql.html" target="_blank">The Curse and Blessings of Dynamic SQL</a>
:   Dynamic-SQL, the technique of generating and execute SQL code on-the-fly is sometimes irreplaceable, sometimes abused, and almost always done in a SQL-injection prone, dangerous, way. This article covers the pros and cons of this technique, when to use and when not, how to use it and what pitfalls to avoid.

Arrays and Lists in SQL Server
:   In lack of a true array type in the Transact-SQL language, developers have resorted to all sort of workarounds to pass sets of data as parameters to Transact-SQL procedures and batches: strings containing comma delimited items, XML variables, temp tables, table value parameters. Erland has a series of articles covering this topic in <a href="http://www.sommarskog.se/arrays-in-sql-2008.html" target="_blank">SQL Server 2008</a> (where table valued parameters are available), <a href="http://www.sommarskog.se/arrays-in-sql-2005.html" target="_blank">SQL Server 2005</a> (where XML data type became available) and <a href="http://www.sommarskog.se/arrays-in-sql-2000.html" target="_blank">SQL Server 2000</a> (the Dark Age).

<a href="http://www.sommarskog.se/share_data.html" target="_blank">How to Share Data Between Stored Procedures</a>
:   All you need to know about how to share data between procedures and how to choose among the various solutions: table valued parameters, table valued functions, temp tables, XML data type, INSERT-EXEC, OPENQUERY, cursor variables, CLR.

Error Handling
:   Another series that covers error handling in Transact-SQL in <a href="http://www.sommarskog.se/error_handling_2005.html" target="_blank">SQL Server 2005</a> and later using TRY-CATCH blocks as well as <a href="http://www.sommarskog.se/error-handling-I.html" target="_blank">SQL Server 2000</a> before TRY-CATCH blocks were available.

<a href="http://www.sommarskog.se/grantperm.html" target="_blank">Giving Permissions through Stored Procedures</a>
:   Code signing is, by a large margin, the perfect way to effectively grant granular control to non-privileged users for specific operations in SQL Server 2005 and later. This powerful mechanism is though seldom used, because of the arcane &#8216;black magic&#8217; that goes into getting the sequence of operations correct to effectively sign a procedure, and because of the complication that arrise from the EXECUTE AS impersonation context required for signed procedures. This article goes to great length in shading some light on this esoteric topic.

<a href="http://www.sommarskog.se/query-plan-mysteries.html" target="_blank">Slow in the Application, Fast in SSMS?</a>
:   This article describes <a href="http://technet.microsoft.com/en-us/library/cc966425.aspx#XSLTsection133121120120" target="_blank">parameter-sniffing</a>, the process of &#8216;tuning&#8217; a query plan to a specific value of the input parameters, and how this process can sometime &#8216;pollute&#8217; the query plan cache with a plan that is optimized for a specific set of parameters, but performs poorly on other parameter values. The article describes what is parameter sniffing, how to identify when it happens, and how to fix it and prevent it.