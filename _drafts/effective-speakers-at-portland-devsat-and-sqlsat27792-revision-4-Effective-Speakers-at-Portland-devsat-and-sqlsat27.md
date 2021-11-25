---
id: 796
title: 'Effective Speakers at Portland #devsat and #sqlsat27'
date: 2010-05-11T22:46:40+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/05/11/792-revision-4/
permalink: /2010/05/11/792-revision-4/
---
Which is faster: <tt>++i</tt> or <tt>i++</tt>?

In 1998 I was looking to change jobs and I interviewed with a company for a C++ developer position. The interview went well and as we were approaching the end, one of the interviewers asked me this question: which is faster ++i or i++? I pondered the question a second, then the other interviewer said that is probably implementation specific, but the first one corrected him that i++ must return the value before the increment therefore it must make a copy of itself, while ++i returns the value after the increment therefore does not need to make a copy of itself, it can return itself. With this my chance to actually answer the question an their chance to see me resolve a problem I didn&#8217;t know was gone, but the interview was finished anyway as we were out of time. I got the offer from them, but ended up in a different place. But that question lingered in my mind, I though what a clever little thing to know. Few months later I got my hands on the <a href="http://www.aristeia.com/books.html" target="_blank">Effective C++</a> book by Scott Meyers, and this opened my appetite for the follow up book More Effective C++. And there it was, item 6 in More Effective C++: _Distinguish between prefix and postfix forms of forms of increment and decrement operators_.

These two books were tremendously important in forming me as a professional C++ developer. They got me starting in studying C++ more deeply, beyond what I had to use in my day to day job. I ended up taking a Brainbench C++ test and I scored in the top 10 worldwide, which pretty soon landed me an email from Microsoft recruiting. The rest, as they say, is history.

SQL Saturday #27 is going to be held on May 22 in Portland and will share the venue with Portland CodeCamp. The list of speakers is really impressive, and amongst them, you guessed, is Scott Meyers presenting <a href="http://portlandcodecamp.org/2010/Sessions/Details/77" target="_blank">CPU Caches and Why You Care</a>. There are many more fine speakers and interesting topics for every taste, and the event is free. Is worth your time if you&#8217;re in the area, and well worth a trip to the City of Roses if you&#8217;re not.

I myself will be presenting a session on <a href="http://sqlsaturday.com/viewsession.aspx?sat=27&#038;sessionid=1708" target="_blank">High Volume Real Time Contiguous ETL and Audit</a>.

to register with the CodeCamp and SQL Saturday http://devsat.eventbrite.com