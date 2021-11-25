---
id: 176
title: SqlDependency.OnChange callback timing
date: 2008-01-04T18:41:18+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/01/04/76-revision/
permalink: /2008/01/04/76-revision/
---
I had reviewed the way SqlDependency works several times in this blog, but the implementation of this feature will probably continue to surprise me for a long time. After reading about numerous reports of ERRORLOG files getting filled with messages like:

<code class="prettyprint lang-sql">The query notification dialog on conversation handle '{5925E62A-A3BA-DC11-9E8E-000C293EC5A4}.'closed due to the following error: '&lt;?xml version="1.0"?&gt;&lt;Error xmlns="http://schemas.microsoft.com/SQL/ServiceBroker/Error"&gt;&lt;Code&gt;-8470&lt;/Code&gt;&lt;Description&gt;Remote service has been dropped.&lt;/Description&gt;&lt;/Error&gt;'.</code>

I decided to take a closer look. <span id="more-75"></span> The immediate cause of this error is quite clear, is it because the target service of a Query Notification subscription was dropped. I have covered the system error 8470 and others in my earlier post [<span style="color: #000000">Resending messages</span>](http://rusanu.com/2007/12/03/resending-messages/). But of course, the real question is why is this happening? Is there some coding error on the part of the application developer and some way to avoid this problem? Is there some configuration issue?

<!--more-->

My usual answer to people complaining about this problem was that they are calling SqlDependency.Stop method too often, thus causing the error. An application should call SqlDependency.Start once when it starts up and SqlDependency.Stop once when it shuts down. Calling it more often would typically be a coding error and the problem can easily be fixed by moving the code that invokes this method into a proper place, perhaps hooking into AppDomain.CurrentDomain.DomainUnload event.

However, it turns out that this is not always the cause. Lets look at how does SqlDependency wait for notifications. Our trusty friend the SQL Profiler reveals the query submitted by the SqlDependency background thread that waits for notifications:

<code class="prettyprint lang-sql">exec sp_executesql N'BEGIN CONVERSATION TIMER (''9c0b82d5-a3ba-dc11-9e8e-000c293ec5a4'') TIMEOUT = 120; WAITFOR(RECEIVE TOP (1) message_type_name, conversation_handle, cast(message_body AS XML) as message_body from [SqlQueryNotificationService-6f91483f-089c-425e-afa6-0c1553ad1b52]), TIMEOUT @p2;',N'@p2 int',@p2=60000</code>

This query will start a conversation timer with a timeout of 2 minutes (120 seconds) and then post a WAITFOR(RECEIVE…) with a timeout of 1 minute (60000 milliseconds). The idea here is that if the application exits abruptly the conversation timer will fire and this will cause the activated procedure attached to the queue to run and this in turn will cleanup the SqlDependency temporary infrastructure (the procedure itself, the service and the queue). Normally the application will not disconnect abruptly so the WAITFOR (RECEIVE) will timeout first after one minute causing the SqlDependency to post back the same query, which will reset the conversation timer again to 2 minutes, so the timer is actually never firing because is continuously moved back 2 minutes.If a notification is received then the WAITFOR(RECEIVE) will dequeue the notification before the 1 minute timeout occurs and, after the application callback is notified, the SqlDependency will again post the same query, resetting the timer again.

 <span class="Apple-style-span" style="font-style: italic">After the application callback is notified…</span> Is there a problem with that? Of course, if the callback is lasting longer than 2 minutes (or more precisely the time left from the original 2 minutes when the query was first launched) then the conversation <span class="Apple-style-span" style="font-weight: bold">will</span> fire and the SqlDependency infrastructure  <span class="Apple-style-span" style="font-weight: bold">will</span> be removed. The only question here is if the callbacks are synchronous or asynchronous. A simple breakpoint on my callback and a look at the stack provides the answer:

<pre style="font-size:10px">SQLNUD.exe!SQLNUD.Program.dep_OnChange(...)
   System.Data.dll!System.Data.SqlClient.SqlDependency.EventContextPair.InvokeCallback(...)
   mscorlib.dll!System.Threading.ExecutionContext.Run(...)
   System.Data.dll!System.Data.SqlClient.SqlDependency.EventContextPair.Invoke(...)
   System.Data.dll!System.Data.SqlClient.SqlDependency.Invalidate(...)
   System.Data.dll!System.Data.SqlClient.SqlDependencyPerAppDomainDispatcher.InvalidateCommandID(...)
   System.Data.dll!SqlDependencyProcessDispatcher.SqlConnectionContainer.ProcessNotificationResults(...)
   System.Data.dll!SqlDependencyProcessDispatcher.SqlConnectionContainer.AsyncResultCallback(...)
   System.Data.dll!System.Data.Common.DbAsyncResult.AsyncCallback_Context(...)
   mscorlib.dll!System.Threading.ExecutionContext.Run(...)
   System.Data.dll!System.Data.Common.DbAsyncResult.ExecuteCallback(...)
   mscorlib.dll!System.Threading._ThreadPoolWaitCallback.WaitCallback_Context(...)
   mscorlib.dll!System.Threading.ExecutionContext.Run(...)
   mscorlib.dll!System.Threading._ThreadPoolWaitCallback.PerformWaitCallbackInternal(...)
   mscorlib.dll!System.Threading._ThreadPoolWaitCallback.PerformWaitCallback(...)</pre>

This stack shows that the application specific callback (the SqlDependency.OnChange) is called synchronously, in the context of processing the WAITFOR(RECEIVE) query results. If this callback exceeds what’s left from the original timer 2 minutes, then the conversation timer will fire and the SqlDependency infrastructure will be removed. One can easily verify this by simply waiting in the debugger on the breakpoint set on the callback. In a short the SQL Profiler will show that the activated procedure was launched and the procedure itself, the service and the queue were dropped. Interestingly enough after the application is resumed the SqlDependency creates a <span class="Apple-style-span" style="font-style: italic">new</span> infrastructure by deploying a new procedure, service and queue.

Of course, 2 minutes to process a callback seems like infinity. But there is one very common scenario that results in such huge times: debugging! When you develop your application you often spend minutes in front of the debugger, inspecting variables and then … discussing with a coworker perhaps, before letting the application resume.

Another thing to consider is that when SqlDependency is used in a normal forms application the callback is usually invoking a form method on the form’s main thread by calling Form.Invoke so that UI elements can be refreshed. The Invoke call will block the callback thread until the UI thread can service the message pump, so if the UI thread is blocked (perhaps in a database call) then the callback thread will also be blocked.

And finally if you simply have a lot of callbacks on the same notification, then the total time to service each callback can add up.

Still 2 minutes is a long time in an application, but if you found yourself faced with investigating mysterious errors showing up in ERRORLOG, one place to look for is this: are you blocking the SqlDependency background thread by any chance?