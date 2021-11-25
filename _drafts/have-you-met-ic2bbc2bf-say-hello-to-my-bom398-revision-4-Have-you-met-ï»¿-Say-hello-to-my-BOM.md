---
id: 403
title: Have you met ï»¿ ? Say hello to my BOM
date: 2009-05-21T23:11:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/05/21/398-revision-4/
permalink: /2009/05/21/398-revision-4/
---
I recently had to look at a problem where a programmer writing a tool to parse some XML produced by a service written by me was complaining that my service serves invalid XML. All my documents started with seemed mangled and started with <div style="color:red>&#239;&#187;&#191;</div> 

. I couldn&#8217;t help but smile. So what is this mysterious sequence ï»¿? Well, lets check the <a href="http://en.wikipedia.org/wiki/ISO_8859-1" target="_blank">ISO-8859-1</a> character encoding, commonly known as &#8216;Latin alphabet&#8217; and often confused with the Windows code page 1252: ï is the character corresponding to 0xEF (decimal 239), » is 0xBB (decimal 187) and ¿ is 0xBF (decimal 191). So the mysterious sequence is 0xEFBBBF. Does it look familiar now? It should, this is the <a href="http://en.wikipedia.org/wiki/Byte-order_mark" target="_blank">UTF-8 Byte Order Mark</a>. Moral: if you consume and parse XML, make sure you consume it as XML, not as text. All XML libraries I know of correctly understand and parse the BOM. The only problems I&#8217;ve seen are from hand written &#8216;parsers&#8217; that treat XML as a string (and most often fail to accommodate namespaces too&#8230;).