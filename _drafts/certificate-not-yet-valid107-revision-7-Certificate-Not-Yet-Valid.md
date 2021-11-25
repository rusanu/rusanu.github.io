---
id: 115
title: Certificate Not Yet Valid
date: 2008-08-25T08:23:45+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/08/25/107-revision-7/
permalink: /2008/08/25/107-revision-7/
---
So you are running into a blocking issue, try to find a solution and you gets back a reply like &#8216;Sorry, I cannot repro your problem&#8217;. You go back and try again, and sure the problem is still there. You dig deeper, go to the Advanced search options on Google, but still cannot find the answer. Desperately, you try again and, surprise, the problem has vanished. Sounds familiar?

I know of one such problem that appears again and again when discussing Service Broker, Database Mirroring and, to a lesser extent, SQL Server 2005 Cryptographic facilities. I want to show you what the problem is, what&#8217;s causing it and even drill a bit into how something so basic has slipped into the product.

<!--more-->

# Certificate Not Yet Valid

Lets do a very simple example of a Service Broker conversation that uses a certificate. First create a test database, then create a service that we will use for our test:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">master</span><span style="color:Black"> </span><span style="color:Blue">key</span><span style="color:Black">
	</span><span style="color:Blue">encryption</span><span style="color:Black"> </span><span style="color:Blue">by</span><span style="color:Black"> </span><span style="color:Blue">password</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'Password#123'</span><span style="color:Gray">;
</span><span style="color:Black">go

</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> test </span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">subject</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'test'</span><span style="color:Gray">;
</span><span style="color:Black">go

</span><span style="color:Blue">backup</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> test </span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'test.cer'</span><span style="color:Gray">;
</span><span style="color:Black">go

</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> testQueue</span><span style="color:Gray">;
</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> testService
	</span><span style="color:Blue">on</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> testQueue </span><span style="color:Gray">(</span><span style="color:Black">[DEFAULT]</span><span style="color:Gray">);
</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">remote</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> </span><span style="color:Blue">binding</span><span style="color:Black"> testBinding
	</span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> </span><span style="color:Red">'testService'</span><span style="color:Black"> </span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">user</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> [dbo]</span><span style="color:Gray">;
</span><span style="color:Black">go</span>
</pre>

This setup allows us to send a message on a conversation from service <span style="color:Green">testService</span> to service <span style="color:Green">testService</span>:

<pre><span style="color: Black"></span><span style="color:Blue">declare</span><span style="color:Black"> @h </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;
</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">dialog</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> [testService]
	</span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> </span><span style="color:Red">'testService'</span><span style="color:Gray">;
</span><span style="color:Blue">send</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h</span><span style="color:Gray">;
</span><span style="color:Black">go</span>
</pre>

And now check the <span style="color:Green">sys.transmission_queue</span>:

<pre><span style="color: Black">
</span><span style="color:Blue">waitfor</span><span style="color:Black"> </span><span style="color:Blue">delay</span><span style="color:Black"> </span><span style="color:Red">'00:00:05'</span><span style="color:Gray">;
</span><span style="color:Blue">select</span><span style="color:Black"> transmission_status </span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.transmission_queue</span><span style="color:Gray">;
</span><span style="color:Black">go</span>
</pre>

And you will see this message: <span style="color:Red">The security certificate bound to database principal (Id: 1) is not yet valid. Either wait for the certificate to become valid or install a certificate that is currently valid</span>. Or you won&#8217;t see anything. Wait! Is there an error or isn&#8217;t there one? Well.. it depends on where you live. And it also depends whether is summer or winter outside.

# Universal Time

Any sort of communication between machines must rely on <a href="http://en.wikipedia.org/wiki/Universal_Time" target="_blank">UTC</a>, because machines can be distributed across the globe and have different local times. Using UTC timestamps solves this issue. This also applies to certificates. The fields <span style="color:Green">Validity/Not Before</span> and <span style="color:Green">Validity/Not After</span> must be expressed in UTC so that the certificate issued on one part of the globe is safely used on another part of the globe. As I write this post from Bucharest that is on the GMT+2 time zone and summer daylight saving time is in effect, my local time is 3 hours ahead of the UTC time. I can test this from SQL:

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Fuchsia">getutcdate</span><span style="color:Gray">(),</span><span style="color:Black"> </span><span style="color:Fuchsia">getdate</span><span style="color:Gray">()</span></pre>

I get back the values <tt>2008-08-25 09:25:16</tt> (my local time) and <tt>2008-08-25 06:25:16</tt> (UTC time). When I created the certificate I did not specify a the <tt>START_DATE</tt> nor the <tt>EXPIRY_DATE</tt> values, so the certificate was created using the creation moment as <span style="color:Green">Validity/Not Before</span> and creation moment + 1 year as <span style="color:Green">Validity/Not After</span>, which are the X.509 fields corresponding to <tt>START_DATE</tt> and <tt>EXPIRY_DATE</tt>. Is there some way to verify this? Yes, we can export the certificate and then open it in the system certificate viewer:

<pre><span style="color: Black"></span><span style="color:Blue">backup</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> test </span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">file</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'test.cer'</span><span style="color:Gray">;
</span><span style="color:Black">go</span></pre>

The <tt>test.cer</tt> file was saved in the same location your <tt>master</tt> files are. Double-click this file and you get to see all the details of the certificate, and we can check the Valid from:

<div class="post-image">
  <a href="http://rusanu.com/wp-content/uploads/2008/08/certificate.PNG" target="_blank"><img src="http://test.rusanu.com/wp-content/uploads/2008/08/certificate.png" alt="Certificate Viewer" title="Click on the image for a full size view" width="250" /></a>
</div>

But you can see that the time displayed is &#8230; <tt>Monday, August 25, 2008 12:25:19 PM</tt>. That is not the 9:25 AM reported by <span style="color:Fuchsia">GETDATE()</span> neither the 6:25 AM reported by <span style="color:Fuchsia">GETUTCDATE()</span>. OK, so my computer local time is 3 hours ahead of UTC and the Certificate Viewer displays the times in local time, that means that the certificate has it&#8217;s UTC time stamp at 9:25. Isn&#8217;t this wrong? It sure is. And in fact that is a SQL Server 2005 bug. When creating a new certificate, it stamps the certificate valid and expiration dates with the local time values instead of the UTC time values. However, when it uses the certificate the time fields are used correctly, so the time value in the field is interpreted as a UTC time value. This is why the certificate I created at 9:25 AM is not valid until 12:25 PM the same day!. And this is because I did the test in Bucharest during summer. If I&#8217;d repeat the same test in Moscow in winter, the certificate would not be valid for 3 hours. In London during summer is not valid one hour. In Mumbai would be 5 hours and 30 minutes. As I said, the result depends on where you&#8217;re doing the test and also whether daylight saving time is in effect or not!

So why some of you did not see any problem? Because if you live on the Western Hemisphere and you are _behind_ the UTC clock, then the certificate is valid immediately as you create it. In fact, is it stamped as being valid few hours in the past, depending on your time zone.

# Redmond, WA, USA

So when you post your problem on the forums, somebody from the SQL team is looking at it, trying to repro, and doesn&#8217;t see any problem. Of course, they are located in Redmond so they don&#8217;t see the problem :). Or perhaps an MVP or a forum moderator is trying to help you. Again, if they are US based (and many are), they would not repro the issue, since their local time is safe from this problem.

Is this problem still in SQL Server 2005? I tested on build 9.00.3042 which is SP2, and the problem is present. How could Microsoft not be aware of this issue when they released the product? Very simple: the entire test infrastructure is running on Redmond time! In fact the problem was reported early by users: <a href="https://connect.microsoft.com/SQLServer/feedback/ViewFeedback.aspx?FeedbackID=125262" target="_blank">https://connect.microsoft.com/SQLServer/feedback/ViewFeedback.aspx?FeedbackID=125262</a> but was closed as not repro. Of course, the person who tried to repro was based on Redmond, where the issue does not repro. Other people are still running into this issue today, look at <a href="http://forums.microsoft.com/msdn/ShowPost.aspx?PostID=3779091&#038;SiteID=1" target="_blank">this</a> post on the MSDN forums, or <a href="http://forums.microsoft.com/MSDN/ShowPost.aspx?PostID=1046928&#038;SiteID=1" target="_blank">here</a>.

# Workaround

How do you resolve the issue? Very simply, specify an explicit <tt>START_DATE</tt> when you create the certificate:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">certificate</span><span style="color:Black"> test
	</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">subject</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'test'</span><span style="color:Gray">,
</span><span style="color:Black">	</span><span style="color:Blue">start_date</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Red">'2008-08-24'</span><span style="color:Gray">;
</span></pre>

Why does the problem seem to sometimes magically fix itself? Very simple, the certificate has become valid. After the number of hours that is the difference between your local time and the UTC time has passed, the certificate is valid.

And one last note: what start date is displayed in the certificate meta data view?

<pre><span style="color: Black"></span><span style="color:Blue">select</span><span style="color:Black"> </span><span style="color:Gray">*</span><span style="color:Black"> </span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Green">sys.certificates</span><span style="color:Gray">
</span></pre>

Yes indeed, that is <tt>2008-08-25 09:25:19.000</tt>. Confusing? You bet. Correct? Not a chance&#8230;