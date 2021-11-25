---
id: 1681
title: Case Sensitive collation sort order
date: 2012-11-22T23:45:14+00:00
author: remus
layout: revision
guid: http://rusanu.com/2012/11/22/1680-revision/
permalink: /2012/11/22/1680-revision/
---
A recent inquiry from one of our front line CSS engineers had me look into how case sensitive collations decide the sort order. Consider a simple question like _How should the values <tt>'a 1'</tt>, <tt>'a 2'</tt>, <tt>'A 1'</tt> and <tt>'A 2'</tt> sort?_

.  
<code