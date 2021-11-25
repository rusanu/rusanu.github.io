---
id: 539
title: Passing Parameters to a Background Procedure
date: 2009-08-17T22:36:50+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/08/17/532-revision-7/
permalink: /2009/08/17/532-revision-7/
---
I have posted previously an example [how to invoke a procedure asynchronously](http://rusanu.com/2009/08/05/asynchronous-procedure-execution) using service Broker activation. Several readers have inquired how to extend this mechanism to add parameters to the background launched procedure.

Passing parameters to a single well know procedure is easy: the parameters are be added to the message body and the activated procedure looks them up in the received XML, passing them to the called procedure. But is significantly more complex to create a generic mechanism that can pass parameters to any procedure. The problem is the type system, because the parameters have unknown types and the activated procedure has to pass proper typed parameters to the invoked procedure.

A generic solution should accept a variety of parameter types and should deal with the peculiarities of Transact-SQL parameters passing, namely the named parameters capabilities. Also the invocation wrapper <tt>usp_AsyncExecInvoke</tt> should directly accept the parameters for the desired background procedure. After considering sevral alternatives, I settled on the following approach:</>

  * Rely on the generic <a href="http://msdn.microsoft.com/en-us/library/ms173829.aspx" target="_blank">sql_variant</a> data type available in SQL Server. The invocation wrapper <tt>usp_AsyncExecInvoke</tt> accept all the parameters as <tt>sql_variant</tt>.
  * Pass the background procedure parameter names explicitly to the invocation wrapper <tt>usp_AsyncExecInvoke</tt>.
  * Always used named parameters in the background procedure invocation.
  * Build a dynamic SQL batch to invoke the background procedure that deals with the required parameter type conversions.

## Accepting Parameters

The invocation wrapper <tt>usp_AsyncExecInvoke<tt> has changed the signature to accept a variable number of parameters:</p> 

<pre>
<span style="color: Black"></span><span style="color:Blue">create procedure </span><span style="color:Black">[usp_AsyncExecInvoke]<br />
    @procedureName </span><span style="color:Blue">sysname<br />
    </span><span style="color:Gray">, </span><span style="color:Black">@p1 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n1 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p2 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n2 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p3 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n3 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p4 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n4 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p5 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n5 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@token </span><span style="color:Blue">uniqueidentifier output</span>
</pre>

<p>
  The new parameters are used to collect the desired parameter values <i>and names</i> to be passed to the background procedure. So sonsider you would like to make the following traditional, synchronous, call:
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">exec </span><span style="color:Black">usp_withParam @id </span><span style="color:Gray">= </span><span style="color:Black">1.0<br />
	</span><span style="color:Gray">, </span><span style="color:Black">@name </span><span style="color:Gray">= </span><span style="color:Black">N</span><span style="color:Red">'Foo'<br />
	</span><span style="color:Gray">, </span><span style="color:Black">@bytes </span><span style="color:Gray">= </span><span style="color:Black">0xBAADF00D</span><span style="color:Gray">;</span>
</pre>

<p>
  The equivalent asynchronous invocation would need to pass the parameter values (1.0, 'Foo' and 0xbaadf00d) as @p1, @p2 and @p3 respectively and the parameter names (@id, @name and @bytes) as @n1, @n2 and @n3: 
  
  <pre>
<span style="color: Black"></span><span style="color:Blue">exec </span><span style="color:Black">usp_AsyncExecInvoke @procedureName </span><span style="color:Gray">= </span><span style="color:Black">N</span><span style="color:Red">'usp_withParam'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p1 </span><span style="color:Gray">= </span><span style="color:Black">1.0</span><span style="color:Gray">, </span><span style="color:Black">@n1 </span><span style="color:Gray">= </span><span style="color:Black">N</span><span style="color:Red">'@id'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p2 </span><span style="color:Gray">= </span><span style="color:Black">N</span><span style="color:Red">'Foo'</span><span style="color:Gray">, </span><span style="color:Black">@n2</span><span style="color:Gray">=</span><span style="color:Red">'@name'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p3 </span><span style="color:Gray">= </span><span style="color:Black">0xBAADF00D</span><span style="color:Gray">, </span><span style="color:Black">@n3 </span><span style="color:Gray">= </span><span style="color:Red">'@bytes'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@token </span><span style="color:Gray">= </span><span style="color:Black">@token </span><span style="color:Blue">output</span><span style="color:Gray">;</span>
</pre>
  
  <p>
    The invocation wrapper will pack the parameter names, values and type information into the message body. This is the XML message body corresponding to the invocation above:
  </p>
  
  <pre>
</pre>
  
  <h2>
    Extracting the Parameters
  </h2>
</p>