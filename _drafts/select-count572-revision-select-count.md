---
id: 573
title: 'select count(*);'
date: 2009-10-26T12:43:10+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/10/26/572-revision/
permalink: /2009/10/26/572-revision/
---
Quick trivia: what is the result of running <code class="prettyprint lang-sql">SELECT COUNT(*);</code>?

That&#8217;s right, no FROM clause, just COUNT(*). The answer may be a little bit surprising, is 1. When you query <code class="prettyprint lang-sql">SELECT 1;</code> the result is, as expected, 1. <code class="prettyprint lang-sql">SELECT 2;</code> will return 2. So <code class="prettyprint lang-sql">SELECT COUNT(2);</code> returns, as expected, 1. But <code class="prettyprint lang-sql">SELECT COUNT(*);</code> has a certain smell of voo-doo to it. Ok, is the * project operator, but project from.. what exactly?

How about <code class="prettyprint lang-sql">SELECT COUNT(*) [MyTable]</code>. Well, that&#8217;s actually just a shortcut for <code class="prettyprint lang-sql">SELECT COUNT(*) AS [MyTable]</code>, so it still returns 1 but in a column named &#8216;MyTable&#8217;. Now you under