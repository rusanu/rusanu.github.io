---
id: 288
title: 'Things I know now: blogging can get you into a ponzi scheme'
date: 2009-03-20T12:01:46+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/03/20/284-revision-4/
permalink: /2009/03/20/284-revision-4/
---
I got tagged by <a href="http://sqlblog.com/blogs/adam_machanic/archive/2009/03/16/things-i-know-now.aspx" target="_blank">Adam Machanic</a>. Although my blog looks like a DBA blog, I am a pure breed developer, so here is what I know now:

### You can&#8217;t fix what you can&#8217;t measure

Successful projects dedicate quite a large amount of resources to instrumentation, profiling and monitoring. Anywhere between 5 and 10% of the used resources should be an acceptable margin to run code that generates logs, reports performance counters, monitors responsiveness, aggregates and consolidates run time data. I cannot stress the importance of properly instrumenting your code to support this. It has been said before that any optimization or troubleshooting should start from measurements, not from guesswork. Your duty is to put those measurements in place, expose information your users can rely on in order to make informed decisions. Don&#8217;t cut corners and <a href="http://en.wikipedia.org/wiki/Eat_one%27s_own_dog_food" target="_blank">eat your dog food</a>: use the instrumentation you added to troubleshoot problems, don&#8217;t fire up a debugger and start stepping through the sources. If you cannot figure out the cause of a problem from the tracing and logs, your customers won&#8217;t be able to do it either. Nor will you be able to troubleshoot an on site customer deployment.

My favorite instrumentation tools are the Performance Counters. I sprinkle them generously in my code. For my own use I have developed XSLT stylesheets I use to quickly generate new performance counters code for any application I write starting from a simple XML definition file. The Windows infrastructure allows me to collect them over time in predefined performance logs. Being able to fire up Perfmon and show how your application performs in real time always impr 

### Your intuition is usually wrong on everything really fast, really big or really small

Your mind is incredibly powerful at analyzing things that happen at everyday pace. But your intuition is most of the times wrong when it comes to things that happen really fast or on a really large scale. There is no chance for that record to change while you released the lock on it? It **will** change under stress. Your application works perfectly on your dual core laptop? Try it on a 64 way machine and you&#8217;ll see that <a href="http://en.wikipedia.org/wiki/Lock_convoy" target="_blank">lock convoys</a> are not a legend, they **do** happen.

###