---
id: 739
title: Passing Parameters to a Background Procedure
date: 2010-03-25T18:39:12+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/25/532-revision-20/
permalink: /2010/03/25/532-revision-20/
---
I have posted previously an example [how to invoke a procedure asynchronously](http://rusanu.com/2009/08/05/asynchronous-procedure-execution) using service Broker activation. Several readers have inquired how to extend this mechanism to add parameters to the background launched procedure.

Passing parameters to a single well know procedure is easy: the parameters are be added to the message body and the activated procedure looks them up in the received XML, passing them to the called procedure. But is significantly more complex to create a generic mechanism that can pass parameters to any procedure. The problem is the type system, because the parameters have unknown types and the activated procedure has to pass proper typed parameters to the invoked procedure.

A generic solution should accept a variety of parameter types and should deal with the peculiarities of Transact-SQL parameters passing, namely the named parameters capabilities. Also the invocation wrapper <tt>usp_AsyncExecInvoke</tt> should directly accept the parameters for the desired background procedure. After considering several alternatives, I settled on the following approach:</>

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
    The invocation wrapper will pack the parameter names, values and type information into the message body. The parameter type information is deduced at run time from the actual parameters, by using the <a href="http://msdn.microsoft.com/en-us/library/ms178550.aspx" target="_blank">SQL_VARIANT_PROPERTY</a> system function. This is the XML message body corresponding to the invocation above (note how the varbinary parameter was encoded using base64 encoding):
  </p>
  
  <pre>
<span style="color: Black"></span><span style="color:Blue">&lt;</span><span style="color:Maroon">procedure</span><span style="color:Blue">&gt;<br />
  &lt;</span><span style="color:Maroon">name</span><span style="color:Blue">&gt;</span><span style="color:Black">usp_withParam</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">name</span><span style="color:Blue">&gt;<br />
  &lt;</span><span style="color:Maroon">parameters</span><span style="color:Blue">&gt;<br />
    &lt;</span><span style="color:Maroon">parameter </span><span style="color:Red">Name</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@id</span><span style="color:Black">" </span><span style="color:Red">BaseType</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">numeric</span><span style="color:Black">" </span><span style="color:Red">Precision</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">2</span><span style="color:Black">" </span><span style="color:Red">Scale</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">1</span><span style="color:Black">" </span><span style="color:Red">MaxLength</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">5</span><span style="color:Black">"</span><span style="color:Blue">&gt;</span><span style="color:Black">1.0</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">parameter</span><span style="color:Blue">&gt;<br />
    &lt;</span><span style="color:Maroon">parameter </span><span style="color:Red">Name</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">" </span><span style="color:Red">BaseType</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">nvarchar</span><span style="color:Black">" </span><span style="color:Red">Precision</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue"></span><span style="color:Black">" </span><span style="color:Red">Scale</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue"></span><span style="color:Black">" </span><span style="color:Red">MaxLength</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">6</span><span style="color:Black">"</span><span style="color:Blue">&gt;</span><span style="color:Black">Foo</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">parameter</span><span style="color:Blue">&gt;<br />
    &lt;</span><span style="color:Maroon">parameter </span><span style="color:Red">Name</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@bytes</span><span style="color:Black">" </span><span style="color:Red">BaseType</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">varbinary</span><span style="color:Black">" </span>
          <span style="color:Red">Precision</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue"></span><span style="color:Black">" </span><span style="color:Red">Scale</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue"></span><span style="color:Black">" </span><span style="color:Red">MaxLength</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">4</span><span style="color:Black">"</span><span style="color:Blue">&gt;</span><span style="color:Black">uq3wDQ==</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">parameter</span><span style="color:Blue">&gt;<br />
  &lt;/</span><span style="color:Maroon">parameters</span><span style="color:Blue">&gt;<br />
&lt;/</span><span style="color:Maroon">procedure</span><span style="color:Blue">&gt;</span>
</pre>
  
  <h2>
    Extracting the Parameters
  </h2>
  
  <p>
    The activated procedure job is now significantly more complex because of the task of extracting the parameter names, types and value and constructing a proper Transact-SQL statement to invoke the original procedure. A new helper procedure was added, <tt>usp_procedureInvokeHelper</tt>, that constructs a dynamic SQL batch from a invokation message body. Here is the dynamic SQL that results from our sample invocation:
  </p>
  
  <pre>
<span style="color: Black"></span><span style="color:Blue">declare </span><span style="color:Black">@pt1 </span><span style="color:Blue">numeric</span><span style="color:Gray">(</span><span style="color:Black">2</span><span style="color:Gray">,</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Blue">declare </span><span style="color:Black">@pt2 </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">6</span><span style="color:Gray">)<br />
</span><span style="color:Blue">declare </span><span style="color:Black">@pt3 </span><span style="color:Blue">varbinary</span><span style="color:Gray">(</span><span style="color:Black">4</span><span style="color:Gray">)<br />
</span><span style="color:Blue">select </span><span style="color:Black">@pt1</span><span style="color:Gray">=</span><span style="color:Black">@x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'(//procedure/parameters/parameter)[1]'</span><span style="color:Gray">, </span><span style="color:Red">N'numeric(2,1)'</span><span style="color:Gray">);<br />
</span><span style="color:Blue">select </span><span style="color:Black">@pt2</span><span style="color:Gray">=</span><span style="color:Black">@x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'(//procedure/parameters/parameter)[2]'</span><span style="color:Gray">, </span><span style="color:Red">N'nvarchar(6)'</span><span style="color:Gray">);<br />
</span><span style="color:Blue">select </span><span style="color:Black">@pt3</span><span style="color:Gray">=</span><span style="color:Black">@x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'(//procedure/parameters/parameter)[3]'</span><span style="color:Gray">, </span><span style="color:Red">N'varbinary(4)'</span><span style="color:Gray">);<br />
</span><span style="color:Blue">exec </span><span style="color:Black">[usp_withParam]  @id</span><span style="color:Gray">=</span><span style="color:Black">@pt1</span><span style="color:Gray">,</span><span style="color:Black">@name</span><span style="color:Gray">=</span><span style="color:Black">@pt2</span><span style="color:Gray">,</span><span style="color:Black">@bytes</span><span style="color:Gray">=</span><span style="color:Black">@pt3<br />
</span>
</pre>
  
  <h2>
    Limitations
  </h2>
  
  <p>
    Because the invocation wrapper uses sql_variant typed parameters, all sql_variant restrictions apply to this sample. Most notably the wrapper will not accept XML type parameters nor large data types like varchar(max) or varbinary(max). It is possible to work around these limitations but it would complicate my sample beyond the point of being usefull as an example how to pass the parameters.
  </p>
  
  <p>
    Because the error handling of the activated procedure will rollback in case of error, passing in a message that results in a procedure invocation error will cause the activation to rollback repeatedly and the poison message detection mechanism will kick in, deactivating the queue. A trivial example is if we omit the required @id parameter for our sample invocation.
  </p>
  
  <h2>
    The code
  </h2>
  
  <p>
    <code class="prettyprint lang-sql">&lt;/p>
&lt;pre>
&lt;span style="color: Black">&lt;/span>&lt;span style="color:Blue">create table &lt;/span>&lt;span style="color:Black">[AsyncExecResults] &lt;/span>&lt;span style="color:Gray">(&lt;br />
 &lt;/span>&lt;span style="color:Black">[token] &lt;/span>&lt;span style="color:Blue">uniqueidentifier primary key&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[submit_time] &lt;/span>&lt;span style="color:Blue">datetime &lt;/span>&lt;span style="color:Gray">not null&lt;br />
 , &lt;/span>&lt;span style="color:Black">[start_time] &lt;/span>&lt;span style="color:Blue">datetime &lt;/span>&lt;span style="color:Gray">null&lt;br />
 , &lt;/span>&lt;span style="color:Black">[finish_time] &lt;/span>&lt;span style="color:Blue">datetime &lt;/span>&lt;span style="color:Gray">null&lt;br />
 , &lt;/span>&lt;span style="color:Black">[error_number] &lt;/span>&lt;span style="color:Blue">int &lt;/span>&lt;span style="color:Gray">null&lt;br />
 , &lt;/span>&lt;span style="color:Black">[error_message] &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">) null);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create queue &lt;/span>&lt;span style="color:Black">[AsyncExecQueue]&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create service &lt;/span>&lt;span style="color:Black">[AsyncExecService] &lt;/span>&lt;span style="color:Blue">on queue &lt;/span>&lt;span style="color:Black">[AsyncExecQueue] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">[DEFAULT]&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">GO&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">-- Dynamic SQL helper procedure&lt;br />
-- Extracts the parameters from the message body&lt;br />
-- Creates the invocation Transact-SQL batch&lt;br />
-- Invokes the dynmic SQL batch&lt;br />
&lt;/span>&lt;span style="color:Blue">create procedure &lt;/span>&lt;span style="color:Black">[usp_procedureInvokeHelper] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@x &lt;/span>&lt;span style="color:Blue">xml&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
begin&lt;br />
    set nocount on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">declare &lt;/span>&lt;span style="color:Black">@stmt &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@stmtDeclarations &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@stmtValues &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@countParams &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@namedParams &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@paramName &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramType &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramPrecision &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramScale &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramLength &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramTypeFull &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">300&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@comma &lt;/span>&lt;span style="color:Blue">nchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@stmtDeclarations &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@stmtValues &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@namedParams &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@comma &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">declare &lt;/span>&lt;span style="color:Black">crsParam &lt;/span>&lt;span style="color:Blue">cursor forward_only static read_only for&lt;br />
        select &lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@Name'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'sysname'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            , &lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@BaseType'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'sysname'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            , &lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@Precision'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'int'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            , &lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@Scale'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'int'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            , &lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@MaxLength'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'int'&lt;/span>&lt;span style="color:Gray">)&lt;br />
        &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">@x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">nodes&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'//procedure/parameters/parameter'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Black">t&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">x&lt;/span>&lt;span style="color:Gray">);&lt;br />
    &lt;/span>&lt;span style="color:Blue">open &lt;/span>&lt;span style="color:Black">crsParam&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">fetch next from &lt;/span>&lt;span style="color:Black">crsParam &lt;/span>&lt;span style="color:Blue">into &lt;/span>&lt;span style="color:Black">@paramName&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramType&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramPrecision&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramScale&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramLength&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">while &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">@@fetch_status &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">)&lt;br />
    &lt;/span>&lt;span style="color:Blue">begin&lt;br />
        select &lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@paramTypeFull &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@paramType &lt;/span>&lt;span style="color:Gray">+&lt;br />
            &lt;/span>&lt;span style="color:Blue">case&lt;br />
            when &lt;/span>&lt;span style="color:Black">@paramType &lt;/span>&lt;span style="color:Gray">in (&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'varchar'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'nvarchar'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'varbinary'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'char'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'nchar'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'binary'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">then&lt;br />
                &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'(' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@paramLength &lt;/span>&lt;span style="color:Blue">as nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">5&lt;/span>&lt;span style="color:Gray">)) + &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">')'&lt;br />
            &lt;/span>&lt;span style="color:Blue">when &lt;/span>&lt;span style="color:Black">@paramType &lt;/span>&lt;span style="color:Gray">in (&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'numeric'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">then&lt;br />
                &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'(' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@paramPrecision &lt;/span>&lt;span style="color:Blue">as nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">10&lt;/span>&lt;span style="color:Gray">)) + &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">',' &lt;/span>&lt;span style="color:Gray">+&lt;br />
                &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@paramScale &lt;/span>&lt;span style="color:Blue">as nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">10&lt;/span>&lt;span style="color:Gray">))+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">')'&lt;br />
            &lt;/span>&lt;span style="color:Blue">else &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;br />
            &lt;/span>&lt;span style="color:Blue">end&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Green">-- Some basic sanity check on the input XML&lt;br />
        &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@paramName &lt;/span>&lt;span style="color:Gray">is NULL&lt;br />
            or &lt;/span>&lt;span style="color:Black">@paramType &lt;/span>&lt;span style="color:Gray">is NULL&lt;br />
            or &lt;/span>&lt;span style="color:Black">@paramTypeFull &lt;/span>&lt;span style="color:Gray">is NULL&lt;br />
            or &lt;/span>&lt;span style="color:Fuchsia">charindex&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''''&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramName&lt;/span>&lt;span style="color:Gray">) &gt; &lt;/span>&lt;span style="color:Black">0&lt;br />
            &lt;/span>&lt;span style="color:Gray">or &lt;/span>&lt;span style="color:Fuchsia">charindex&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''''&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramTypeFull&lt;/span>&lt;span style="color:Gray">) &gt; &lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Incorrect parameter attributes %i: %s:%s %i:%i:%i'&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">10&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@i&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramName&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramType&lt;br />
                &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramPrecision&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramScale&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramLength&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@stmtDeclarations &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@stmtDeclarations &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'&lt;br />
declare @pt' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Blue">as varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">3&lt;/span>&lt;span style="color:Gray">)) + &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">' ' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@paramTypeFull&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@stmtValues &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@stmtValues &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'&lt;br />
select @pt' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Blue">as varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">3&lt;/span>&lt;span style="color:Gray">)) + &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'=@x.value(&lt;br />
    N''(//procedure/parameters/parameter)[' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Blue">as varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">3&lt;/span>&lt;span style="color:Gray">)) &lt;br />
                + &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">']'', N''' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@paramTypeFull &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Red">''');'&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@namedParams &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@namedParams &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@comma &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@paramName&lt;br />
                &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'=@pt' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">cast&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@i &lt;/span>&lt;span style="color:Blue">as varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">3&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@comma &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">','&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">fetch next from &lt;/span>&lt;span style="color:Black">crsParam &lt;/span>&lt;span style="color:Blue">into &lt;/span>&lt;span style="color:Black">@paramName&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramType&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramPrecision&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramScale&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@paramLength&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;br />
    close &lt;/span>&lt;span style="color:Black">crsParam&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">deallocate &lt;/span>&lt;span style="color:Black">crsParam&lt;/span>&lt;span style="color:Gray">;        &lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@stmt &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@stmtDeclarations &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@stmtValues &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'&lt;br />
exec ' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Fuchsia">quotename&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@x&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'(//procedure/name)[1]'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'sysname'&lt;/span>&lt;span style="color:Gray">));&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@namedParams &lt;/span>&lt;span style="color:Gray">!= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">''&lt;/span>&lt;span style="color:Gray">)&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@stmt &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@stmt &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">' ' &lt;/span>&lt;span style="color:Gray">+ &lt;/span>&lt;span style="color:Black">@namedParams&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">exec &lt;/span>&lt;span style="color:Maroon">sp_executesql &lt;/span>&lt;span style="color:Black">@stmt&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@x xml'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@x&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create procedure &lt;/span>&lt;span style="color:Black">usp_AsyncExecActivated&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
begin&lt;br />
    set nocount on&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">declare &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Blue">xml&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@startTime &lt;/span>&lt;span style="color:Blue">datetime&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@finishTime &lt;/span>&lt;span style="color:Blue">datetime&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorNumber &lt;/span>&lt;span style="color:Blue">int&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorMessage &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">)&lt;br />
        , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Blue">smallint&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
    &lt;/span>&lt;span style="color:Blue">begin transaction&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">begin try&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">receive top&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_handle]&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[message_type_name]&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[message_body]&lt;br />
            &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">[AsyncExecQueue]&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Gray">is not null)&lt;br />
        &lt;/span>&lt;span style="color:Blue">begin&lt;br />
            if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'DEFAULT'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                &lt;/span>&lt;span style="color:Green">-- The DEFAULT message type is a procedure invocation.&lt;br />
                --&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">CAST&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">as xml&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;br />
                &lt;/span>&lt;span style="color:Blue">save transaction &lt;/span>&lt;span style="color:Black">usp_AsyncExec_procedure&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@startTime &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">GETUTCDATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                &lt;/span>&lt;span style="color:Blue">begin try&lt;br />
                    exec &lt;/span>&lt;span style="color:Black">[usp_procedureInvokeHelper] @xmlBody&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">end try&lt;br />
                begin catch&lt;br />
                &lt;/span>&lt;span style="color:Green">-- This catch block tries to deal with failures of the procedure execution&lt;br />
                -- If possible it rolls back to the savepoint created earlier, allowing&lt;br />
                -- the activated procedure to continue. If the executed procedure&lt;br />
                -- raises an error with severity 16 or higher, it will doom the transaction&lt;br />
                -- and thus rollback the RECEIVE. Such case will be a poison message,&lt;br />
                -- resulting in the queue disabling.&lt;br />
                --&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@execErrorNumber &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_NUMBER&lt;/span>&lt;span style="color:Gray">(),&lt;br />
                    &lt;/span>&lt;span style="color:Black">@execErrorMessage &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_MESSAGE&lt;/span>&lt;span style="color:Gray">(),&lt;br />
                    &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">XACT_STATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= -&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
                &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                    rollback&lt;/span>&lt;span style="color:Gray">;&lt;br />
                    &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Unrecoverable error in procedure: %i: %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">10&lt;/span>&lt;span style="color:Gray">,&lt;br />
                        &lt;/span>&lt;span style="color:Black">@execErrorNumber&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@execErrorMessage&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Blue">end&lt;br />
                else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
                &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                    rollback transaction &lt;/span>&lt;span style="color:Black">usp_AsyncExec_procedure&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">end&lt;br />
                end catch&lt;br />
&lt;br />
                select &lt;/span>&lt;span style="color:Black">@finishTime &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">GETUTCDATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_id]&lt;br />
                    &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys.conversation_endpoints&lt;br />
                    &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[conversation_handle] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">is null)&lt;br />
                &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                    raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Internal consistency error: conversation not found'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">20&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Blue">end&lt;br />
                update &lt;/span>&lt;span style="color:Black">[AsyncExecResults] &lt;/span>&lt;span style="color:Blue">set&lt;br />
                    &lt;/span>&lt;span style="color:Black">[start_time] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@starttime&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[finish_time] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@finishTime&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_number] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@execErrorNumber&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_message] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@execErrorMessage&lt;br />
                    &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[token] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@token&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">0 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">@@ROWCOUNT&lt;/span>&lt;span style="color:Gray">)&lt;br />
                &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                    raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Internal consistency error: token not found'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">30&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Blue">end&lt;br />
                end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
            &lt;/span>&lt;span style="color:Blue">end&lt;br />
            else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
            &lt;/span>&lt;span style="color:Blue">end&lt;br />
            else if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageTypeName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Blue">begin&lt;br />
                declare &lt;/span>&lt;span style="color:Black">@errorNumber &lt;/span>&lt;span style="color:Blue">int&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@errorMessage &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">4000&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">CAST&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@messageBody &lt;/span>&lt;span style="color:Blue">as xml&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Blue">with xmlnamespaces &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Blue">DEFAULT &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'http://schemas.microsoft.com/SQL/ServiceBroker/Error'&lt;/span>&lt;span style="color:Gray">)&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@errorNumber &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'(/Error/Code)[1]'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'INT'&lt;/span>&lt;span style="color:Gray">),&lt;br />
                    &lt;/span>&lt;span style="color:Black">@errorMessage &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">value &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'(/Error/Description)[1]'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'NVARCHAR(4000)'&lt;/span>&lt;span style="color:Gray">);&lt;br />
                &lt;/span>&lt;span style="color:Green">-- Update the request with the received error&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_id] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys.conversation_endpoints &lt;br />
                    &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[conversation_handle] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">update &lt;/span>&lt;span style="color:Black">[AsyncExecResults] &lt;/span>&lt;span style="color:Blue">set&lt;br />
                    &lt;/span>&lt;span style="color:Black">[start_time] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@starttime&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_number] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@errorNumber&lt;br />
                    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[error_message] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@errorMessage&lt;br />
                    &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[token] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@token&lt;/span>&lt;span style="color:Gray">;&lt;br />
                &lt;/span>&lt;span style="color:Blue">end conversation &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
             &lt;/span>&lt;span style="color:Blue">end&lt;br />
           else&lt;br />
           begin&lt;br />
                raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Received unexpected message type: %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">50&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@messageTypeName&lt;/span>&lt;span style="color:Gray">);&lt;br />
           &lt;/span>&lt;span style="color:Blue">end&lt;br />
        end&lt;br />
        commit&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">end try&lt;br />
    begin catch&lt;br />
        declare &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Blue">int&lt;br />
         &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">);&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_NUMBER&lt;/span>&lt;span style="color:Gray">()&lt;br />
            , &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_MESSAGE&lt;/span>&lt;span style="color:Gray">()&lt;br />
            , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">XACT_STATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
        &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">&lt;&gt; &lt;/span>&lt;span style="color:Black">0&lt;/span>&lt;span style="color:Gray">)&lt;br />
        &lt;/span>&lt;span style="color:Blue">begin&lt;br />
         rollback&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">end&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Error: %i, %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">60&lt;/span>&lt;span style="color:Gray">,  &lt;/span>&lt;span style="color:Black">@error&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">with &lt;/span>&lt;span style="color:Fuchsia">log&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">end catch&lt;br />
end&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">alter queue &lt;/span>&lt;span style="color:Black">[AsyncExecQueue]&lt;br />
    &lt;/span>&lt;span style="color:Blue">with &lt;/span>&lt;span style="color:Black">activation &lt;/span>&lt;span style="color:Gray">(&lt;br />
    &lt;/span>&lt;span style="color:Black">procedure_name &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[usp_AsyncExecActivated]&lt;br />
    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">max_queue_readers &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1&lt;br />
    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">execute as owner&lt;br />
    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">status &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Blue">on&lt;/span>&lt;span style="color:Gray">);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">-- Helper function to create the XML element &lt;br />
-- for a passed in parameter&lt;br />
&lt;/span>&lt;span style="color:Blue">create function &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;br />
 &lt;/span>&lt;span style="color:Black">@p &lt;/span>&lt;span style="color:Blue">sql_variant&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n &lt;/span>&lt;span style="color:Blue">sysname&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">returns xml&lt;br />
with schemabinding&lt;br />
as&lt;br />
begin&lt;br />
 return &lt;/span>&lt;span style="color:Gray">(&lt;br />
 &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@n &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[@Name]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Fuchsia">sql_variant_property&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'BaseType'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[@BaseType]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Fuchsia">sql_variant_property&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'Precision'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[@Precision]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Fuchsia">sql_variant_property&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'Scale'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[@Scale]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Fuchsia">sql_variant_property&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'MaxLength'&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[@MaxLength]&lt;br />
  &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@p&lt;br />
  &lt;/span>&lt;span style="color:Blue">for xml path&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'parameter'&lt;/span>&lt;span style="color:Gray">), &lt;/span>&lt;span style="color:Blue">type&lt;/span>&lt;span style="color:Gray">)&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;/span>&lt;span style="color:Black">GO&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">-- Invocation wrapper. Accepts arbitrary&lt;br />
-- named parameetrs to be passed to the&lt;br />
-- background procedure&lt;br />
&lt;/span>&lt;span style="color:Blue">create procedure &lt;/span>&lt;span style="color:Black">[usp_AsyncExecInvoke]&lt;br />
    @procedureName &lt;/span>&lt;span style="color:Blue">sysname&lt;br />
    &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@p1 &lt;/span>&lt;span style="color:Blue">sql_variant &lt;/span>&lt;span style="color:Gray">= NULL, &lt;/span>&lt;span style="color:Black">@n1 &lt;/span>&lt;span style="color:Blue">sysname &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">@p2 &lt;/span>&lt;span style="color:Blue">sql_variant &lt;/span>&lt;span style="color:Gray">= NULL, &lt;/span>&lt;span style="color:Black">@n2 &lt;/span>&lt;span style="color:Blue">sysname &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">@p3 &lt;/span>&lt;span style="color:Blue">sql_variant &lt;/span>&lt;span style="color:Gray">= NULL, &lt;/span>&lt;span style="color:Black">@n3 &lt;/span>&lt;span style="color:Blue">sysname &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">@p4 &lt;/span>&lt;span style="color:Blue">sql_variant &lt;/span>&lt;span style="color:Gray">= NULL, &lt;/span>&lt;span style="color:Black">@n4 &lt;/span>&lt;span style="color:Blue">sysname &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">@p5 &lt;/span>&lt;span style="color:Blue">sql_variant &lt;/span>&lt;span style="color:Gray">= NULL, &lt;/span>&lt;span style="color:Black">@n5 &lt;/span>&lt;span style="color:Blue">sysname &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Blue">uniqueidentifier output&lt;br />
as&lt;br />
begin&lt;br />
    declare &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;br />
     &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Blue">xml&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Blue">int&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">set nocount on&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
 &lt;/span>&lt;span style="color:Blue">set &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">@@trancount&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;br />
        &lt;/span>&lt;span style="color:Blue">begin transaction&lt;br />
    else&lt;br />
        save transaction &lt;/span>&lt;span style="color:Black">usp_AsyncExecInvoke&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">begin try&lt;br />
        begin dialog conversation &lt;/span>&lt;span style="color:Black">@h&lt;br />
            &lt;/span>&lt;span style="color:Blue">from service &lt;/span>&lt;span style="color:Black">[AsyncExecService]&lt;br />
            &lt;/span>&lt;span style="color:Blue">to service &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'AsyncExecService'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Red">'current database'&lt;br />
            &lt;/span>&lt;span style="color:Blue">with encryption &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Blue">off&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">[conversation_id]&lt;br />
            &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Green">sys.conversation_endpoints&lt;br />
            &lt;/span>&lt;span style="color:Blue">where &lt;/span>&lt;span style="color:Black">[conversation_handle] &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@h&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@xmlBody &lt;/span>&lt;span style="color:Gray">= (&lt;br />
            &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@procedureName &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[name]&lt;br />
            &lt;/span>&lt;span style="color:Gray">, (&lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Gray">* &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Gray">(&lt;br />
                &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p1&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n1&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">AS &lt;/span>&lt;span style="color:Black">[*] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">WHERE &lt;/span>&lt;span style="color:Black">@p1 &lt;/span>&lt;span style="color:Gray">IS NOT NULL&lt;br />
                &lt;/span>&lt;span style="color:Blue">union all select &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p2&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n2&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">AS &lt;/span>&lt;span style="color:Black">[*] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">WHERE &lt;/span>&lt;span style="color:Black">@p2 &lt;/span>&lt;span style="color:Gray">IS NOT NULL&lt;br />
                &lt;/span>&lt;span style="color:Blue">union all select &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p3&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n3&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">AS &lt;/span>&lt;span style="color:Black">[*] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">WHERE &lt;/span>&lt;span style="color:Black">@p3 &lt;/span>&lt;span style="color:Gray">IS NOT NULL&lt;br />
                &lt;/span>&lt;span style="color:Blue">union all select &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p4&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n4&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">AS &lt;/span>&lt;span style="color:Black">[*] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">WHERE &lt;/span>&lt;span style="color:Black">@p4 &lt;/span>&lt;span style="color:Gray">IS NOT NULL&lt;br />
                &lt;/span>&lt;span style="color:Blue">union all select &lt;/span>&lt;span style="color:Black">[dbo]&lt;/span>&lt;span style="color:Gray">.&lt;/span>&lt;span style="color:Black">[fn_DescribeSqlVariant] &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@p5&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n5&lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">AS &lt;/span>&lt;span style="color:Black">[*] &lt;br />
                    &lt;/span>&lt;span style="color:Blue">WHERE &lt;/span>&lt;span style="color:Black">@p5 &lt;/span>&lt;span style="color:Gray">IS NOT NULL&lt;br />
                ) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">p &lt;/span>&lt;span style="color:Blue">for xml path&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">''&lt;/span>&lt;span style="color:Gray">), &lt;/span>&lt;span style="color:Blue">type&lt;br />
            &lt;/span>&lt;span style="color:Gray">) &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[parameters]&lt;br />
            &lt;/span>&lt;span style="color:Blue">for xml path&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Red">'procedure'&lt;/span>&lt;span style="color:Gray">), &lt;/span>&lt;span style="color:Blue">type&lt;/span>&lt;span style="color:Gray">);&lt;br />
        &lt;/span>&lt;span style="color:Blue">send on conversation &lt;/span>&lt;span style="color:Black">@h &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@xmlBody&lt;/span>&lt;span style="color:Gray">);&lt;br />
        &lt;/span>&lt;span style="color:Blue">insert into &lt;/span>&lt;span style="color:Black">[AsyncExecResults]&lt;br />
            &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">[token]&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">[submit_time]&lt;/span>&lt;span style="color:Gray">)&lt;br />
            &lt;/span>&lt;span style="color:Blue">values&lt;br />
            &lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">@token&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Fuchsia">getutcdate&lt;/span>&lt;span style="color:Gray">());&lt;br />
    &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;br />
        &lt;/span>&lt;span style="color:Blue">commit&lt;/span>&lt;span style="color:Gray">;&lt;br />
    &lt;/span>&lt;span style="color:Blue">end try&lt;br />
    begin catch&lt;br />
        declare &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Blue">int&lt;br />
            &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Blue">nvarchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">2048&lt;/span>&lt;span style="color:Gray">)&lt;br />
            , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Blue">smallint&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@error &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_NUMBER&lt;/span>&lt;span style="color:Gray">()&lt;br />
            , &lt;/span>&lt;span style="color:Black">@message &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">ERROR_MESSAGE&lt;/span>&lt;span style="color:Gray">()&lt;br />
            , &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Fuchsia">XACT_STATE&lt;/span>&lt;span style="color:Gray">();&lt;br />
        &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= -&lt;/span>&lt;span style="color:Black">1&lt;br />
            &lt;/span>&lt;span style="color:Blue">rollback&lt;/span>&lt;span style="color:Gray">;&lt;br />
        &lt;/span>&lt;span style="color:Blue">if &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1 &lt;/span>&lt;span style="color:Gray">and &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;br />
            &lt;/span>&lt;span style="color:Blue">rollback&lt;br />
        if &lt;/span>&lt;span style="color:Black">@xactState &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1 &lt;/span>&lt;span style="color:Gray">and &lt;/span>&lt;span style="color:Black">@trancount &lt;/span>&lt;span style="color:Gray">&gt; &lt;/span>&lt;span style="color:Black">0&lt;br />
            &lt;/span>&lt;span style="color:Blue">rollback transaction &lt;/span>&lt;span style="color:Black">usp_my_procedure_name&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
        &lt;/span>&lt;span style="color:Blue">raiserror&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Error: %i, %s'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">16&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@error&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@message&lt;/span>&lt;span style="color:Gray">);&lt;br />
    &lt;/span>&lt;span style="color:Blue">end catch&lt;br />
end&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Green">-- Sample invocation example&lt;br />
-- The usp_withParam will insert &lt;br />
-- all the received parameters into this table&lt;br />
-- &lt;br />
&lt;/span>&lt;span style="color:Blue">create table &lt;/span>&lt;span style="color:Black">[withParam] &lt;/span>&lt;span style="color:Gray">(&lt;br />
    &lt;/span>&lt;span style="color:Black">id &lt;/span>&lt;span style="color:Blue">numeric&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">4&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">) NULL&lt;br />
    , &lt;/span>&lt;span style="color:Blue">name varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">150&lt;/span>&lt;span style="color:Gray">) NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">date &lt;/span>&lt;span style="color:Blue">datetime  &lt;/span>&lt;span style="color:Gray">NULL&lt;br />
    , &lt;/span>&lt;span style="color:Blue">value int &lt;/span>&lt;span style="color:Gray">NULL&lt;br />
    , &lt;/span>&lt;span style="color:Black">bytes &lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">) NULL);&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">create procedure &lt;/span>&lt;span style="color:Black">usp_withParam&lt;br />
 @id &lt;/span>&lt;span style="color:Blue">numeric&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">4&lt;/span>&lt;span style="color:Gray">,&lt;/span>&lt;span style="color:Black">1&lt;/span>&lt;span style="color:Gray">)&lt;br />
 , &lt;/span>&lt;span style="color:Black">@name &lt;/span>&lt;span style="color:Blue">varchar&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Black">150&lt;/span>&lt;span style="color:Gray">)&lt;br />
 , &lt;/span>&lt;span style="color:Black">@date &lt;/span>&lt;span style="color:Blue">datetime &lt;/span>&lt;span style="color:Gray">= NULL&lt;br />
 , &lt;/span>&lt;span style="color:Black">@value &lt;/span>&lt;span style="color:Blue">int &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@bytes &lt;/span>&lt;span style="color:Blue">varbinary&lt;/span>&lt;span style="color:Gray">(&lt;/span>&lt;span style="color:Fuchsia">max&lt;/span>&lt;span style="color:Gray">) = NULL&lt;br />
&lt;/span>&lt;span style="color:Blue">as&lt;br />
begin&lt;br />
    insert into &lt;/span>&lt;span style="color:Black">[withParam] &lt;/span>&lt;span style="color:Gray">(&lt;br />
        &lt;/span>&lt;span style="color:Black">id&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">name&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">date&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Blue">value&lt;br />
        &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">bytes&lt;/span>&lt;span style="color:Gray">)&lt;br />
     &lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Black">@id &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[id]&lt;br />
      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@name &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[name]&lt;br />
      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@date &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[date]&lt;br />
      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@value &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[value]&lt;br />
      &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@bytes &lt;/span>&lt;span style="color:Blue">as &lt;/span>&lt;span style="color:Black">[bytes]&lt;br />
&lt;/span>&lt;span style="color:Blue">end&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">declare &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Blue">uniqueidentifier&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">exec &lt;/span>&lt;span style="color:Black">usp_AsyncExecInvoke @procedureName &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'usp_withParam'&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@p1 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">1.0&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n1 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'@id'&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@p2 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">N&lt;/span>&lt;span style="color:Red">'Foo'&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n2&lt;/span>&lt;span style="color:Gray">=&lt;/span>&lt;span style="color:Red">'@name'&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@p3 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">0xBAADF00D&lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@n3 &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Red">'@bytes'&lt;br />
 &lt;/span>&lt;span style="color:Gray">, &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Gray">= &lt;/span>&lt;span style="color:Black">@token &lt;/span>&lt;span style="color:Blue">output&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">waitfor delay &lt;/span>&lt;span style="color:Red">'00:00:05'&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Gray">* &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">AsyncExecResults&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;/span>&lt;span style="color:Blue">select &lt;/span>&lt;span style="color:Gray">* &lt;/span>&lt;span style="color:Blue">from &lt;/span>&lt;span style="color:Black">withParam&lt;/span>&lt;span style="color:Gray">;&lt;br />
&lt;br />
&lt;/span>&lt;span style="color:Black">go&lt;br />
&lt;/span>
&lt;/pre>
&lt;p></code>
  </p>