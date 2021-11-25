---
id: 547
title: Passing Parameters to a Background Procedure
date: 2009-08-18T11:18:36+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/08/18/532-revision-15/
permalink: /2009/08/18/532-revision-15/
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
    Because the invocation wrapper uses sql_variant typed parameters, all sql_variant restrictions apply to this sample. Most notably the wrapper will not accept XML type paramaters nor large data types like varchar(max) or varbinary(max). It is possible to work around these limitations but it would complicate my sample beyond the point of being usefull as an example how to pass the parameters.
  </p>
  
  <p>
    Because the error handling of the activated procedure will rollback in case of error, passing in a message that results in a procedure invocation error will cause the activation to rollback repeatedly and the poison message detection mechanism will kick in, deactivating the queue. A trivial example is if we omit the required @id parameter for our sample invocation.
  </p>
  
  <h2>
    The code
  </h2>
  
  <pre>
<span style="color: Black"></span><span style="color:Blue">create table </span><span style="color:Black">[AsyncExecResults] </span><span style="color:Gray">(<br />
 </span><span style="color:Black">[token] </span><span style="color:Blue">uniqueidentifier primary key<br />
 </span><span style="color:Gray">, </span><span style="color:Black">[submit_time] </span><span style="color:Blue">datetime </span><span style="color:Gray">not null<br />
 , </span><span style="color:Black">[start_time] </span><span style="color:Blue">datetime </span><span style="color:Gray">null<br />
 , </span><span style="color:Black">[finish_time] </span><span style="color:Blue">datetime </span><span style="color:Gray">null<br />
 , </span><span style="color:Black">[error_number] </span><span style="color:Blue">int </span><span style="color:Gray">null<br />
 , </span><span style="color:Black">[error_message] </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">) null);<br />
</span><span style="color:Blue">go<br />
<br />
create queue </span><span style="color:Black">[AsyncExecQueue]</span><span style="color:Gray">;<br />
</span><span style="color:Blue">go<br />
<br />
create service </span><span style="color:Black">[AsyncExecService] </span><span style="color:Blue">on queue </span><span style="color:Black">[AsyncExecQueue] </span><span style="color:Gray">(</span><span style="color:Black">[DEFAULT]</span><span style="color:Gray">);<br />
</span><span style="color:Blue">GO<br />
<br />
</span><span style="color:Green">-- Dynamic SQL helper procedure<br />
-- Extracts the parameters from the message body<br />
-- Creates the invocation Transact-SQL batch<br />
-- Invokes the dynmic SQL batch<br />
</span><span style="color:Blue">create procedure </span><span style="color:Black">[usp_procedureInvokeHelper] </span><span style="color:Gray">(</span><span style="color:Black">@x </span><span style="color:Blue">xml</span><span style="color:Gray">)<br />
</span><span style="color:Blue">as<br />
begin<br />
    set nocount on</span><span style="color:Gray">;<br />
    <br />
    </span><span style="color:Blue">declare </span><span style="color:Black">@stmt </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@stmtDeclarations </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@stmtValues </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@i </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@countParams </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@namedParams </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@paramName </span><span style="color:Blue">sysname<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramType </span><span style="color:Blue">sysname<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramPrecision </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramScale </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramLength </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramTypeFull </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">300</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@comma </span><span style="color:Blue">nchar</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">)<br />
        <br />
    </span><span style="color:Blue">select </span><span style="color:Black">@i </span><span style="color:Gray">= </span><span style="color:Black"><br />
        </span><span style="color:Gray">, </span><span style="color:Black">@stmtDeclarations </span><span style="color:Gray">= </span><span style="color:Red">N''<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@stmtValues </span><span style="color:Gray">= </span><span style="color:Red">N''<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@namedParams </span><span style="color:Gray">= </span><span style="color:Red">N''<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@comma </span><span style="color:Gray">= </span><span style="color:Red">N''<br />
<br />
    </span><span style="color:Blue">declare </span><span style="color:Black">crsParam </span><span style="color:Blue">cursor forward_only static read_only for<br />
        select </span><span style="color:Black">x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'@Name'</span><span style="color:Gray">, </span><span style="color:Red">N'sysname'</span><span style="color:Gray">)<br />
            , </span><span style="color:Black">x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'@BaseType'</span><span style="color:Gray">, </span><span style="color:Red">N'sysname'</span><span style="color:Gray">)<br />
            , </span><span style="color:Black">x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'@Precision'</span><span style="color:Gray">, </span><span style="color:Red">N'int'</span><span style="color:Gray">)<br />
            , </span><span style="color:Black">x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'@Scale'</span><span style="color:Gray">, </span><span style="color:Red">N'int'</span><span style="color:Gray">)<br />
            , </span><span style="color:Black">x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'@MaxLength'</span><span style="color:Gray">, </span><span style="color:Red">N'int'</span><span style="color:Gray">)<br />
        </span><span style="color:Blue">from </span><span style="color:Black">@x</span><span style="color:Gray">.</span><span style="color:Black">nodes</span><span style="color:Gray">(</span><span style="color:Red">N'//procedure/parameters/parameter'</span><span style="color:Gray">) </span><span style="color:Black">t</span><span style="color:Gray">(</span><span style="color:Black">x</span><span style="color:Gray">);<br />
    </span><span style="color:Blue">open </span><span style="color:Black">crsParam</span><span style="color:Gray">;<br />
    <br />
    </span><span style="color:Blue">fetch next from </span><span style="color:Black">crsParam </span><span style="color:Blue">into </span><span style="color:Black">@paramName<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramType<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramPrecision<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramScale<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@paramLength</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">while </span><span style="color:Gray">(</span><span style="color:Fuchsia">@@fetch_status </span><span style="color:Gray">= </span><span style="color:Black"></span><span style="color:Gray">)<br />
    </span><span style="color:Blue">begin<br />
        select </span><span style="color:Black">@i </span><span style="color:Gray">= </span><span style="color:Black">@i </span><span style="color:Gray">+ </span><span style="color:Black">1</span><span style="color:Gray">;<br />
<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@paramTypeFull </span><span style="color:Gray">= </span><span style="color:Black">@paramType </span><span style="color:Gray">+<br />
            </span><span style="color:Blue">case<br />
            when </span><span style="color:Black">@paramType </span><span style="color:Gray">in (</span><span style="color:Red">N'varchar'<br />
                </span><span style="color:Gray">, </span><span style="color:Red">N'nvarchar'<br />
                </span><span style="color:Gray">, </span><span style="color:Red">N'varbinary'<br />
                </span><span style="color:Gray">, </span><span style="color:Red">N'char'<br />
                </span><span style="color:Gray">, </span><span style="color:Red">N'nchar'<br />
                </span><span style="color:Gray">, </span><span style="color:Red">N'binary'</span><span style="color:Gray">) </span><span style="color:Blue">then<br />
                </span><span style="color:Red">N'(' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@paramLength </span><span style="color:Blue">as nvarchar</span><span style="color:Gray">(</span><span style="color:Black">5</span><span style="color:Gray">)) + </span><span style="color:Red">N')'<br />
            </span><span style="color:Blue">when </span><span style="color:Black">@paramType </span><span style="color:Gray">in (</span><span style="color:Red">N'numeric'</span><span style="color:Gray">) </span><span style="color:Blue">then<br />
                </span><span style="color:Red">N'(' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@paramPrecision </span><span style="color:Blue">as nvarchar</span><span style="color:Gray">(</span><span style="color:Black">10</span><span style="color:Gray">)) + </span><span style="color:Red">N',' </span><span style="color:Gray">+<br />
                </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@paramScale </span><span style="color:Blue">as nvarchar</span><span style="color:Gray">(</span><span style="color:Black">10</span><span style="color:Gray">))+ </span><span style="color:Red">N')'<br />
            </span><span style="color:Blue">else </span><span style="color:Red">N''<br />
            </span><span style="color:Blue">end</span><span style="color:Gray">;<br />
        <br />
        </span><span style="color:Green">-- Some basic sanity check on the input XML<br />
        </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@paramName </span><span style="color:Gray">is NULL<br />
            or </span><span style="color:Black">@paramType </span><span style="color:Gray">is NULL<br />
            or </span><span style="color:Black">@paramTypeFull </span><span style="color:Gray">is NULL<br />
            or </span><span style="color:Fuchsia">charindex</span><span style="color:Gray">(</span><span style="color:Red">N''''</span><span style="color:Gray">, </span><span style="color:Black">@paramName</span><span style="color:Gray">) &gt; </span><span style="color:Black"><br />
            </span><span style="color:Gray">or </span><span style="color:Fuchsia">charindex</span><span style="color:Gray">(</span><span style="color:Red">N''''</span><span style="color:Gray">, </span><span style="color:Black">@paramTypeFull</span><span style="color:Gray">) &gt; </span><span style="color:Black"></span><span style="color:Gray">)<br />
            </span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Incorrect parameter attributes %i: %s:%s %i:%i:%i'<br />
                </span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">10</span><span style="color:Gray">, </span><span style="color:Black">@i</span><span style="color:Gray">, </span><span style="color:Black">@paramName</span><span style="color:Gray">, </span><span style="color:Black">@paramType<br />
                </span><span style="color:Gray">, </span><span style="color:Black">@paramPrecision</span><span style="color:Gray">, </span><span style="color:Black">@paramScale</span><span style="color:Gray">, </span><span style="color:Black">@paramLength</span><span style="color:Gray">);<br />
        <br />
        </span><span style="color:Blue">select </span><span style="color:Black">@stmtDeclarations </span><span style="color:Gray">= </span><span style="color:Black">@stmtDeclarations </span><span style="color:Gray">+ </span><span style="color:Red">N'<br />
declare @pt' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@i </span><span style="color:Blue">as varchar</span><span style="color:Gray">(</span><span style="color:Black">3</span><span style="color:Gray">)) + </span><span style="color:Red">N' ' </span><span style="color:Gray">+ </span><span style="color:Black">@paramTypeFull<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@stmtValues </span><span style="color:Gray">= </span><span style="color:Black">@stmtValues </span><span style="color:Gray">+ </span><span style="color:Red">N'<br />
select @pt' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@i </span><span style="color:Blue">as varchar</span><span style="color:Gray">(</span><span style="color:Black">3</span><span style="color:Gray">)) + </span><span style="color:Red">N'=@x.value(<br />
    N''(//procedure/parameters/parameter)[' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@i </span><span style="color:Blue">as varchar</span><span style="color:Gray">(</span><span style="color:Black">3</span><span style="color:Gray">)) <br />
                + </span><span style="color:Red">N']'', N''' </span><span style="color:Gray">+ </span><span style="color:Black">@paramTypeFull </span><span style="color:Gray">+ </span><span style="color:Red">''');'<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@namedParams </span><span style="color:Gray">= </span><span style="color:Black">@namedParams </span><span style="color:Gray">+ </span><span style="color:Black">@comma </span><span style="color:Gray">+ </span><span style="color:Black">@paramName<br />
                </span><span style="color:Gray">+ </span><span style="color:Red">N'=@pt' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@i </span><span style="color:Blue">as varchar</span><span style="color:Gray">(</span><span style="color:Black">3</span><span style="color:Gray">));<br />
            <br />
        </span><span style="color:Blue">select </span><span style="color:Black">@comma </span><span style="color:Gray">= </span><span style="color:Red">N','</span><span style="color:Gray">;<br />
        <br />
        </span><span style="color:Blue">fetch next from </span><span style="color:Black">crsParam </span><span style="color:Blue">into </span><span style="color:Black">@paramName<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@paramType<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@paramPrecision<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@paramScale<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@paramLength</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">end<br />
    <br />
    close </span><span style="color:Black">crsParam</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">deallocate </span><span style="color:Black">crsParam</span><span style="color:Gray">;        <br />
<br />
    </span><span style="color:Blue">select </span><span style="color:Black">@stmt </span><span style="color:Gray">= </span><span style="color:Black">@stmtDeclarations </span><span style="color:Gray">+ </span><span style="color:Black">@stmtValues </span><span style="color:Gray">+ </span><span style="color:Red">N'<br />
exec ' </span><span style="color:Gray">+ </span><span style="color:Fuchsia">quotename</span><span style="color:Gray">(</span><span style="color:Black">@x</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">N'(//procedure/name)[1]'</span><span style="color:Gray">, </span><span style="color:Red">N'sysname'</span><span style="color:Gray">));<br />
    <br />
    </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@namedParams </span><span style="color:Gray">!= </span><span style="color:Red">N''</span><span style="color:Gray">)<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@stmt </span><span style="color:Gray">= </span><span style="color:Black">@stmt </span><span style="color:Gray">+ </span><span style="color:Red">N' ' </span><span style="color:Gray">+ </span><span style="color:Black">@namedParams</span><span style="color:Gray">;<br />
<br />
    </span><span style="color:Blue">exec </span><span style="color:Maroon">sp_executesql </span><span style="color:Black">@stmt</span><span style="color:Gray">, </span><span style="color:Red">N'@x xml'</span><span style="color:Gray">, </span><span style="color:Black">@x</span><span style="color:Gray">;<br />
</span><span style="color:Blue">end<br />
go<br />
<br />
create procedure </span><span style="color:Black">usp_AsyncExecActivated<br />
</span><span style="color:Blue">as<br />
begin<br />
    set nocount on</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">declare </span><span style="color:Black">@h </span><span style="color:Blue">uniqueidentifier<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@messageTypeName </span><span style="color:Blue">sysname<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@messageBody </span><span style="color:Blue">varbinary</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@xmlBody </span><span style="color:Blue">xml<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@startTime </span><span style="color:Blue">datetime<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@finishTime </span><span style="color:Blue">datetime<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@execErrorNumber </span><span style="color:Blue">int<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@execErrorMessage </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">)<br />
        , </span><span style="color:Black">@xactState </span><span style="color:Blue">smallint<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@token </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;<br />
<br />
    </span><span style="color:Blue">begin transaction</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">begin try</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">receive top</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">)<br />
            </span><span style="color:Black">@h </span><span style="color:Gray">= </span><span style="color:Black">[conversation_handle]<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@messageTypeName </span><span style="color:Gray">= </span><span style="color:Black">[message_type_name]<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@messageBody </span><span style="color:Gray">= </span><span style="color:Black">[message_body]<br />
            </span><span style="color:Blue">from </span><span style="color:Black">[AsyncExecQueue]</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@h </span><span style="color:Gray">is not null)<br />
        </span><span style="color:Blue">begin<br />
            if </span><span style="color:Gray">(</span><span style="color:Black">@messageTypeName </span><span style="color:Gray">= </span><span style="color:Red">N'DEFAULT'</span><span style="color:Gray">)<br />
            </span><span style="color:Blue">begin<br />
                </span><span style="color:Green">-- The DEFAULT message type is a procedure invocation.<br />
                --<br />
                </span><span style="color:Blue">select </span><span style="color:Black">@xmlBody </span><span style="color:Gray">= </span><span style="color:Fuchsia">CAST</span><span style="color:Gray">(</span><span style="color:Black">@messageBody </span><span style="color:Blue">as xml</span><span style="color:Gray">);<br />
<br />
                </span><span style="color:Blue">save transaction </span><span style="color:Black">usp_AsyncExec_procedure</span><span style="color:Gray">;<br />
                </span><span style="color:Blue">select </span><span style="color:Black">@startTime </span><span style="color:Gray">= </span><span style="color:Fuchsia">GETUTCDATE</span><span style="color:Gray">();<br />
                </span><span style="color:Blue">begin try<br />
                    exec </span><span style="color:Black">[usp_procedureInvokeHelper] @xmlBody</span><span style="color:Gray">;<br />
                </span><span style="color:Blue">end try<br />
                begin catch<br />
                </span><span style="color:Green">-- This catch block tries to deal with failures of the procedure execution<br />
                -- If possible it rolls back to the savepoint created earlier, allowing<br />
                -- the activated procedure to continue. If the executed procedure<br />
                -- raises an error with severity 16 or higher, it will doom the transaction<br />
                -- and thus rollback the RECEIVE. Such case will be a poison message,<br />
                -- resulting in the queue disabling.<br />
                --<br />
                </span><span style="color:Blue">select </span><span style="color:Black">@execErrorNumber </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">(),<br />
                    </span><span style="color:Black">@execErrorMessage </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">(),<br />
                    </span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();<br />
                </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@xactState </span><span style="color:Gray">= -</span><span style="color:Black">1</span><span style="color:Gray">)<br />
                </span><span style="color:Blue">begin<br />
                    rollback</span><span style="color:Gray">;<br />
                    </span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Unrecoverable error in procedure: %i: %s'</span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">10</span><span style="color:Gray">,<br />
                        </span><span style="color:Black">@execErrorNumber</span><span style="color:Gray">, </span><span style="color:Black">@execErrorMessage</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">end<br />
                else if </span><span style="color:Gray">(</span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Black">1</span><span style="color:Gray">)<br />
                </span><span style="color:Blue">begin<br />
                    rollback transaction </span><span style="color:Black">usp_AsyncExec_procedure</span><span style="color:Gray">;<br />
                </span><span style="color:Blue">end<br />
                end catch<br />
<br />
                select </span><span style="color:Black">@finishTime </span><span style="color:Gray">= </span><span style="color:Fuchsia">GETUTCDATE</span><span style="color:Gray">();<br />
                </span><span style="color:Blue">select </span><span style="color:Black">@token </span><span style="color:Gray">= </span><span style="color:Black">[conversation_id]<br />
                    </span><span style="color:Blue">from </span><span style="color:Green">sys</span><span style="color:Gray">.</span><span style="color:Green">conversation_endpoints<br />
                    </span><span style="color:Blue">where </span><span style="color:Black">[conversation_handle] </span><span style="color:Gray">= </span><span style="color:Black">@h</span><span style="color:Gray">;<br />
                </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@token </span><span style="color:Gray">is null)<br />
                </span><span style="color:Blue">begin<br />
                    raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Internal consistency error: conversation not found'</span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">20</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">end<br />
                update </span><span style="color:Black">[AsyncExecResults] </span><span style="color:Blue">set<br />
                    </span><span style="color:Black">[start_time] </span><span style="color:Gray">= </span><span style="color:Black">@starttime<br />
                    </span><span style="color:Gray">, </span><span style="color:Black">[finish_time] </span><span style="color:Gray">= </span><span style="color:Black">@finishTime<br />
                    </span><span style="color:Gray">, </span><span style="color:Black">[error_number] </span><span style="color:Gray">= </span><span style="color:Black">@execErrorNumber<br />
                    </span><span style="color:Gray">, </span><span style="color:Black">[error_message] </span><span style="color:Gray">= </span><span style="color:Black">@execErrorMessage<br />
                    </span><span style="color:Blue">where </span><span style="color:Black">[token] </span><span style="color:Gray">= </span><span style="color:Black">@token</span><span style="color:Gray">;<br />
                </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">0 </span><span style="color:Gray">= </span><span style="color:Fuchsia">@@ROWCOUNT</span><span style="color:Gray">)<br />
                </span><span style="color:Blue">begin<br />
                    raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Internal consistency error: token not found'</span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">30</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">end<br />
                end conversation </span><span style="color:Black">@h</span><span style="color:Gray">;<br />
            </span><span style="color:Blue">end<br />
            else if </span><span style="color:Gray">(</span><span style="color:Black">@messageTypeName </span><span style="color:Gray">= </span><span style="color:Red">N'http://schemas.microsoft.com/SQL/ServiceBroker/EndDialog'</span><span style="color:Gray">)<br />
            </span><span style="color:Blue">begin<br />
                end conversation </span><span style="color:Black">@h</span><span style="color:Gray">;<br />
            </span><span style="color:Blue">end<br />
            else if </span><span style="color:Gray">(</span><span style="color:Black">@messageTypeName </span><span style="color:Gray">= </span><span style="color:Red">N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'</span><span style="color:Gray">)<br />
            </span><span style="color:Blue">begin<br />
                declare </span><span style="color:Black">@errorNumber </span><span style="color:Blue">int<br />
                    </span><span style="color:Gray">, </span><span style="color:Black">@errorMessage </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">4000</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">with </span><span style="color:Black">xmlnamespaces </span><span style="color:Gray">(</span><span style="color:Blue">DEFAULT </span><span style="color:Red">N'http://schemas.microsoft.com/SQL/ServiceBroker/Error'</span><span style="color:Gray">)<br />
                </span><span style="color:Blue">select </span><span style="color:Black">@errorNumber </span><span style="color:Gray">= </span><span style="color:Black">@xmlBody</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'(/Error/Code)[1]'</span><span style="color:Gray">, </span><span style="color:Red">'INT'</span><span style="color:Gray">),<br />
                    </span><span style="color:Black">@errorMessage </span><span style="color:Gray">= </span><span style="color:Black">@xmlBody</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'(/Error/Description)[1]'</span><span style="color:Gray">, </span><span style="color:Red">'NVARCHAR(4000)'</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'A conversation error was encountered: %i: %s'</span><span style="color:Gray">, </span><span style="color:Black">1</span><span style="color:Gray">, </span><span style="color:Black">40</span><span style="color:Gray">,<br />
                    </span><span style="color:Black">@errorNumber</span><span style="color:Gray">, </span><span style="color:Black">@errorMessage</span><span style="color:Gray">);<br />
                </span><span style="color:Blue">end conversation </span><span style="color:Black">@h</span><span style="color:Gray">;<br />
           </span><span style="color:Blue">end<br />
           else<br />
           begin<br />
                raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Received unexpected message type: %s'</span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">50</span><span style="color:Gray">, </span><span style="color:Black">@messageTypeName</span><span style="color:Gray">);<br />
           </span><span style="color:Blue">end<br />
        end<br />
        commit</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">end try<br />
    begin catch<br />
        declare </span><span style="color:Black">@error </span><span style="color:Blue">int<br />
         </span><span style="color:Gray">, </span><span style="color:Black">@message </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">);<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@error </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">()<br />
            , </span><span style="color:Black">@message </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">()<br />
            , </span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();<br />
        </span><span style="color:Blue">if </span><span style="color:Gray">(</span><span style="color:Black">@xactState </span><span style="color:Gray">&lt;&gt; </span><span style="color:Black"></span><span style="color:Gray">)<br />
        </span><span style="color:Blue">begin<br />
         rollback</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">end</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Error: %i, %s'</span><span style="color:Gray">, </span><span style="color:Black">1</span><span style="color:Gray">, </span><span style="color:Black">60</span><span style="color:Gray">,  </span><span style="color:Black">@error</span><span style="color:Gray">, </span><span style="color:Black">@message</span><span style="color:Gray">) </span><span style="color:Blue">with </span><span style="color:Fuchsia">log</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">end catch<br />
end<br />
go<br />
<br />
alter queue </span><span style="color:Black">[AsyncExecQueue]<br />
    </span><span style="color:Blue">with activation </span><span style="color:Gray">(<br />
    </span><span style="color:Blue">procedure_name </span><span style="color:Gray">= </span><span style="color:Black">[usp_AsyncExecActivated]<br />
    </span><span style="color:Gray">, </span><span style="color:Blue">max_queue_readers </span><span style="color:Gray">= </span><span style="color:Black">1<br />
    </span><span style="color:Gray">, </span><span style="color:Blue">execute as owner<br />
    </span><span style="color:Gray">, </span><span style="color:Blue">status </span><span style="color:Gray">= </span><span style="color:Blue">on</span><span style="color:Gray">);<br />
</span><span style="color:Blue">go<br />
<br />
</span><span style="color:Green">-- Helper function to create the XML element <br />
-- for a passed in parameter<br />
</span><span style="color:Blue">create function </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(<br />
 </span><span style="color:Black">@p </span><span style="color:Blue">sql_variant<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@n </span><span style="color:Blue">sysname</span><span style="color:Gray">)<br />
</span><span style="color:Blue">returns xml<br />
with schemabinding<br />
as<br />
begin<br />
 return </span><span style="color:Gray">(<br />
 </span><span style="color:Blue">select </span><span style="color:Black">@n </span><span style="color:Blue">as </span><span style="color:Black">[@Name]<br />
  </span><span style="color:Gray">, </span><span style="color:Fuchsia">sql_variant_property</span><span style="color:Gray">(</span><span style="color:Black">@p</span><span style="color:Gray">, </span><span style="color:Red">'BaseType'</span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[@BaseType]<br />
  </span><span style="color:Gray">, </span><span style="color:Fuchsia">sql_variant_property</span><span style="color:Gray">(</span><span style="color:Black">@p</span><span style="color:Gray">, </span><span style="color:Red">'Precision'</span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[@Precision]<br />
  </span><span style="color:Gray">, </span><span style="color:Fuchsia">sql_variant_property</span><span style="color:Gray">(</span><span style="color:Black">@p</span><span style="color:Gray">, </span><span style="color:Red">'Scale'</span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[@Scale]<br />
  </span><span style="color:Gray">, </span><span style="color:Fuchsia">sql_variant_property</span><span style="color:Gray">(</span><span style="color:Black">@p</span><span style="color:Gray">, </span><span style="color:Red">'MaxLength'</span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[@MaxLength]<br />
  </span><span style="color:Gray">, </span><span style="color:Black">@p<br />
  </span><span style="color:Blue">for xml path</span><span style="color:Gray">(</span><span style="color:Red">'parameter'</span><span style="color:Gray">), </span><span style="color:Blue">type</span><span style="color:Gray">)<br />
</span><span style="color:Blue">end<br />
GO<br />
<br />
</span><span style="color:Green">-- Invocation wrapper. Accepts arbitrary<br />
-- named parameetrs to be passed to the<br />
-- background procedure<br />
</span><span style="color:Blue">create procedure </span><span style="color:Black">[usp_AsyncExecInvoke]<br />
    @procedureName </span><span style="color:Blue">sysname<br />
    </span><span style="color:Gray">, </span><span style="color:Black">@p1 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n1 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p2 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n2 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p3 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n3 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p4 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n4 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@p5 </span><span style="color:Blue">sql_variant </span><span style="color:Gray">= NULL, </span><span style="color:Black">@n5 </span><span style="color:Blue">sysname </span><span style="color:Gray">= NULL<br />
    , </span><span style="color:Black">@token </span><span style="color:Blue">uniqueidentifier output<br />
as<br />
begin<br />
    declare </span><span style="color:Black">@h </span><span style="color:Blue">uniqueidentifier<br />
     </span><span style="color:Gray">, </span><span style="color:Black">@xmlBody </span><span style="color:Blue">xml<br />
        </span><span style="color:Gray">, </span><span style="color:Black">@trancount </span><span style="color:Blue">int</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">set nocount on</span><span style="color:Gray">;<br />
<br />
 </span><span style="color:Blue">set </span><span style="color:Black">@trancount </span><span style="color:Gray">= </span><span style="color:Fuchsia">@@trancount</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">if </span><span style="color:Black">@trancount </span><span style="color:Gray">= </span><span style="color:Black"><br />
        </span><span style="color:Blue">begin transaction<br />
    else<br />
        save transaction </span><span style="color:Black">usp_AsyncExecInvoke</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">begin try<br />
        begin </span><span style="color:Black">dialog </span><span style="color:Blue">conversation </span><span style="color:Black">@h<br />
            </span><span style="color:Blue">from service </span><span style="color:Black">[AsyncExecService]<br />
            </span><span style="color:Blue">to service </span><span style="color:Red">N'AsyncExecService'</span><span style="color:Gray">, </span><span style="color:Red">'current database'<br />
            </span><span style="color:Blue">with encryption </span><span style="color:Gray">= </span><span style="color:Blue">off</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@token </span><span style="color:Gray">= </span><span style="color:Black">[conversation_id]<br />
            </span><span style="color:Blue">from </span><span style="color:Green">sys</span><span style="color:Gray">.</span><span style="color:Green">conversation_endpoints<br />
            </span><span style="color:Blue">where </span><span style="color:Black">[conversation_handle] </span><span style="color:Gray">= </span><span style="color:Black">@h</span><span style="color:Gray">;<br />
<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@xmlBody </span><span style="color:Gray">= (<br />
            </span><span style="color:Blue">select </span><span style="color:Black">@procedureName </span><span style="color:Blue">as </span><span style="color:Black">[name]<br />
            </span><span style="color:Gray">, (</span><span style="color:Blue">select </span><span style="color:Gray">* </span><span style="color:Blue">from </span><span style="color:Gray">(<br />
                </span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p1</span><span style="color:Gray">, </span><span style="color:Black">@n1</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] <br />
                    </span><span style="color:Blue">WHERE </span><span style="color:Black">@p1 </span><span style="color:Gray">IS NOT NULL<br />
                </span><span style="color:Blue">union </span><span style="color:Gray">all </span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p2</span><span style="color:Gray">, </span><span style="color:Black">@n2</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] <br />
                    </span><span style="color:Blue">WHERE </span><span style="color:Black">@p2 </span><span style="color:Gray">IS NOT NULL<br />
                </span><span style="color:Blue">union </span><span style="color:Gray">all </span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p3</span><span style="color:Gray">, </span><span style="color:Black">@n3</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] <br />
                    </span><span style="color:Blue">WHERE </span><span style="color:Black">@p3 </span><span style="color:Gray">IS NOT NULL<br />
                </span><span style="color:Blue">union </span><span style="color:Gray">all </span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p4</span><span style="color:Gray">, </span><span style="color:Black">@n4</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] <br />
                    </span><span style="color:Blue">WHERE </span><span style="color:Black">@p4 </span><span style="color:Gray">IS NOT NULL<br />
                </span><span style="color:Blue">union </span><span style="color:Gray">all </span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p5</span><span style="color:Gray">, </span><span style="color:Black">@n5</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] <br />
                    </span><span style="color:Blue">WHERE </span><span style="color:Black">@p5 </span><span style="color:Gray">IS NOT NULL<br />
                ) </span><span style="color:Blue">as </span><span style="color:Black">p </span><span style="color:Blue">for xml path</span><span style="color:Gray">(</span><span style="color:Red">''</span><span style="color:Gray">), </span><span style="color:Blue">type<br />
            </span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[parameters]<br />
            </span><span style="color:Blue">for xml path</span><span style="color:Gray">(</span><span style="color:Red">'procedure'</span><span style="color:Gray">), </span><span style="color:Blue">type</span><span style="color:Gray">);<br />
        </span><span style="color:Blue">send on conversation </span><span style="color:Black">@h </span><span style="color:Gray">(</span><span style="color:Black">@xmlBody</span><span style="color:Gray">);<br />
        </span><span style="color:Blue">insert into </span><span style="color:Black">[AsyncExecResults]<br />
            </span><span style="color:Gray">(</span><span style="color:Black">[token]</span><span style="color:Gray">, </span><span style="color:Black">[submit_time]</span><span style="color:Gray">)<br />
            </span><span style="color:Blue">values<br />
            </span><span style="color:Gray">(</span><span style="color:Black">@token</span><span style="color:Gray">, </span><span style="color:Fuchsia">getutcdate</span><span style="color:Gray">());<br />
    </span><span style="color:Blue">if </span><span style="color:Black">@trancount </span><span style="color:Gray">= </span><span style="color:Black"><br />
        </span><span style="color:Blue">commit</span><span style="color:Gray">;<br />
    </span><span style="color:Blue">end try<br />
    begin catch<br />
        declare </span><span style="color:Black">@error </span><span style="color:Blue">int<br />
            </span><span style="color:Gray">, </span><span style="color:Black">@message </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Black">2048</span><span style="color:Gray">)<br />
            , </span><span style="color:Black">@xactState </span><span style="color:Blue">smallint</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">select </span><span style="color:Black">@error </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">()<br />
            , </span><span style="color:Black">@message </span><span style="color:Gray">= </span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">()<br />
            , </span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();<br />
        </span><span style="color:Blue">if </span><span style="color:Black">@xactState </span><span style="color:Gray">= -</span><span style="color:Black">1<br />
            </span><span style="color:Blue">rollback</span><span style="color:Gray">;<br />
        </span><span style="color:Blue">if </span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Black">1 </span><span style="color:Gray">and </span><span style="color:Black">@trancount </span><span style="color:Gray">= </span><span style="color:Black"><br />
            </span><span style="color:Blue">rollback<br />
        if </span><span style="color:Black">@xactState </span><span style="color:Gray">= </span><span style="color:Black">1 </span><span style="color:Gray">and </span><span style="color:Black">@trancount </span><span style="color:Gray">&gt; </span><span style="color:Black"><br />
            </span><span style="color:Blue">rollback transaction </span><span style="color:Black">usp_my_procedure_name</span><span style="color:Gray">;<br />
<br />
        </span><span style="color:Blue">raiserror</span><span style="color:Gray">(</span><span style="color:Red">N'Error: %i, %s'</span><span style="color:Gray">, </span><span style="color:Black">16</span><span style="color:Gray">, </span><span style="color:Black">1</span><span style="color:Gray">, </span><span style="color:Black">@error</span><span style="color:Gray">, </span><span style="color:Black">@message</span><span style="color:Gray">);<br />
    </span><span style="color:Blue">end catch<br />
end<br />
go<br />
<br />
</span><span style="color:Green">-- Sample invocation example<br />
-- The usp_withParam will insert <br />
-- all the received parameters into this table<br />
-- <br />
</span><span style="color:Blue">create table </span><span style="color:Black">[withParam] </span><span style="color:Gray">(<br />
    </span><span style="color:Black">id </span><span style="color:Blue">numeric</span><span style="color:Gray">(</span><span style="color:Black">4</span><span style="color:Gray">,</span><span style="color:Black">1</span><span style="color:Gray">) NULL<br />
    , </span><span style="color:Black">name </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">150</span><span style="color:Gray">) NULL<br />
    , </span><span style="color:Blue">date datetime  </span><span style="color:Gray">NULL<br />
    , </span><span style="color:Black">value </span><span style="color:Blue">int </span><span style="color:Gray">NULL<br />
    , </span><span style="color:Black">bytes </span><span style="color:Blue">varbinary</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">) NULL);<br />
</span><span style="color:Blue">go<br />
<br />
create procedure </span><span style="color:Black">usp_withParam<br />
 @id </span><span style="color:Blue">numeric</span><span style="color:Gray">(</span><span style="color:Black">4</span><span style="color:Gray">,</span><span style="color:Black">1</span><span style="color:Gray">)<br />
 , </span><span style="color:Black">@name </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">150</span><span style="color:Gray">)<br />
 , </span><span style="color:Black">@date </span><span style="color:Blue">datetime </span><span style="color:Gray">= NULL<br />
 , </span><span style="color:Black">@value </span><span style="color:Blue">int </span><span style="color:Gray">= </span><span style="color:Black"><br />
 </span><span style="color:Gray">, </span><span style="color:Black">@bytes </span><span style="color:Blue">varbinary</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">) = NULL<br />
</span><span style="color:Blue">as<br />
begin<br />
    insert into </span><span style="color:Black">[withParam] </span><span style="color:Gray">(<br />
        </span><span style="color:Black">id<br />
        </span><span style="color:Gray">, </span><span style="color:Black">name<br />
        </span><span style="color:Gray">, </span><span style="color:Blue">date<br />
        </span><span style="color:Gray">, </span><span style="color:Black">value<br />
        </span><span style="color:Gray">, </span><span style="color:Black">bytes</span><span style="color:Gray">)<br />
     </span><span style="color:Blue">select </span><span style="color:Black">@id </span><span style="color:Blue">as </span><span style="color:Black">[id]<br />
      </span><span style="color:Gray">, </span><span style="color:Black">@name </span><span style="color:Blue">as </span><span style="color:Black">[name]<br />
      </span><span style="color:Gray">, </span><span style="color:Black">@date </span><span style="color:Blue">as </span><span style="color:Black">[date]<br />
      </span><span style="color:Gray">, </span><span style="color:Black">@value </span><span style="color:Blue">as </span><span style="color:Black">[value]<br />
      </span><span style="color:Gray">, </span><span style="color:Black">@bytes </span><span style="color:Blue">as </span><span style="color:Black">[bytes]<br />
</span><span style="color:Blue">end<br />
go<br />
<br />
declare </span><span style="color:Black">@token </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Blue">exec </span><span style="color:Black">usp_AsyncExecInvoke @procedureName </span><span style="color:Gray">= </span><span style="color:Red">N'usp_withParam'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p1 </span><span style="color:Gray">= </span><span style="color:Black">1.0</span><span style="color:Gray">, </span><span style="color:Black">@n1 </span><span style="color:Gray">= </span><span style="color:Red">N'@id'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p2 </span><span style="color:Gray">= </span><span style="color:Red">N'Foo'</span><span style="color:Gray">, </span><span style="color:Black">@n2</span><span style="color:Gray">=</span><span style="color:Red">'@name'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@p3 </span><span style="color:Gray">= </span><span style="color:Black">0xBAADF00D</span><span style="color:Gray">, </span><span style="color:Black">@n3 </span><span style="color:Gray">= </span><span style="color:Red">'@bytes'<br />
 </span><span style="color:Gray">, </span><span style="color:Black">@token </span><span style="color:Gray">= </span><span style="color:Black">@token </span><span style="color:Blue">output</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Blue">waitfor delay </span><span style="color:Red">'00:00:05'</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Blue">select </span><span style="color:Gray">* </span><span style="color:Blue">from </span><span style="color:Black">AsyncExecResults</span><span style="color:Gray">;<br />
</span><span style="color:Blue">select </span><span style="color:Gray">* </span><span style="color:Blue">from </span><span style="color:Black">withParam</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Blue">go</span>
</pre>