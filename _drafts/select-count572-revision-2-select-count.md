---
id: 574
title: 'select count(*);'
date: 2009-10-26T12:45:48+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/10/26/572-revision-2/
permalink: /2009/10/26/572-revision-2/
---
Quick trivia: what is the result of running <code class="prettyprint lang-sql">SELECT COUNT(*);</code>?

That&#8217;s right, no <code class="prettyprint lang-sql">FROM</code> clause, just <code class="prettyprint lang-sql">COUNT(*)</code>. The answer may be a little bit surprising, is <bb>1</bb>. When you query <code class="prettyprint lang-sql">SELECT 1;</code> the result is, as expected, 1. And <code class="prettyprint lang-sql">SELECT 2;</code> will return 2. So <code class="prettyprint lang-sql">SELECT COUNT(2);</code> returns, as expected, 1. But <code class="prettyprint lang-sql">SELECT COUNT(*);</code> has a certain smell of voo-doo to it. Ok, is the * project operator, but project from&#8230; what exactly? It feels like a count is materialized out of the blue and

How about <code class="prettyprint lang-sql">SELECT COUNT(*) [MyTable]</code>. Well, that&#8217;s actually just a shortcut for <code class="prettyprint lang-sql">SELECT COUNT(*) AS [MyTable]</code>, so it still returns 1 but in a column named &#8216;MyTable&#8217;. Now you understan why my hart missed a bit when I checked how I iniitalized a relication subscription and I forgot to type in **FROM**&#8230;