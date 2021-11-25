---
id: 295
title: 'Things I know now: blogging can get you into a email ponzi scheme'
date: 2009-03-20T12:29:49+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/03/20/284-revision-11/
permalink: /2009/03/20/284-revision-11/
---
I got tagged by <a href="http://sqlblog.com/blogs/adam_machanic/archive/2009/03/16/things-i-know-now.aspx" target="_blank">Adam Machanic</a>. Although my blog looks like a DBA blog, I am a pure breed developer, so here is what I know now:

### You can&#8217;t fix what you can&#8217;t measure

Successful projects dedicate quite a large amount of resources to instrumentation, profiling and monitoring. Anywhere between 5 and 10% of the used resources should be an acceptable margin to run code that generates logs, reports performance counters, monitors responsiveness, aggregates and consolidates run time data. I cannot stress the importance of properly instrumenting your code to support this. It has been said before that any optimization or troubleshooting should start from measurements, not from guesswork. Your duty is to put those measurements in place, expose information your users can rely on in order to make informed decisions. Don&#8217;t cut corners and <a href="http://en.wikipedia.org/wiki/Eat_one%27s_own_dog_food" target="_blank">eat your dog food</a>: use the instrumentation you added to troubleshoot problems, don&#8217;t fire up a debugger and start stepping through the sources. If you cannot figure out the cause of a problem from the tracing and logs, your customers won&#8217;t be able to do it either. Nor will you be able to troubleshoot an on site customer deployment.

My favorite instrumentation tools are the Performance Counters. I sprinkle them generously in my code. For my own use I have developed XSLT stylesheets I use to quickly generate new performance counters code for any application I write starting from a simple XML definition file. The Windows infrastructure allows me to collect them over time in predefined performance logs. Being able to fire up Perfmon and show how your application performs in real time always impresses your customers.

### If is not tested, it doesn&#8217;t work correctly

I am yet to see a piece of code that was bug free from the first iteration. Testing is always required to validate that the development team correctly understood the requirements, that the assumptions made by the code author where actually correct, that the requirements correctly captured the end user needs. Besides this functional testing, the stress testing never ever fails to produce bugs. Corner cases, exception code paths and unexpected error returns are always guaranteed to reveal bugs and defects in your code.

Testing requires a different mind set than the development work. Seldom is a good developer a good tester too. I have witnessed often how embracing a <a href="http://en.wikipedia.org/wiki/Test-driven_development" target="_blank">TDD</a> style is confused with actually testing your product. Unfortunately this is not the case. Besides the obvious case of the fox guarding the hen house that is developers writing tests for their own code, the unit tests produced by TDD are not equivalent to the end-to-end use case testing that is required to validate that your product is actually working and doing what is expected to do.

### Your intuition is often wrong

Your mind is incredibly powerful at analyzing things that happen at everyday pace. But your intuition is most of the times wrong when it comes to things that happen really fast or on a really large scale. There is no chance for that record to change while you released the lock on it? It **will** change under stress. Your application works perfectly on your dual core laptop? Try it on a 64 way machine and you&#8217;ll see that <a href="http://en.wikipedia.org/wiki/Lock_convoy" target="_blank">lock convoys</a> are not a legend, they **do** happen.

Writing high scale and high performance programs is a league of its own. Most techniques you learn prior to attempting something really big don&#8217;t work. Everything is different: memory allocation, thread management, how you do your I/O. Fortunately there are some excellent articles covering these topics and my favorite ones are the ones from <a href="http://rusanu.com/2008/11/11/high-performance-windows-programs/" target="_blank"Rick Vicik</a>.

Who&#8217;s am I tagging? I&#8217;m only gonna choose one person, thus in violation of this game rules, but he is a very special person and if I can get him back into the blogosphere, is well worth it: <a href="http://blogs.msdn.com/slavao/" target="_blank">Slava</a>