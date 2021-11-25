---
id: 533
title: Passing Parameters to a Background Procedure
date: 2009-08-17T22:21:12+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/08/17/532-revision/
permalink: /2009/08/17/532-revision/
---
I have posted previously an example [how to invoke a procedure asynchronously](http://rusanu.com/2009/08/05/asynchronous-procedure-execution) using service Broker activation. Several readers have inquired how to extend this mechanism to add parameters to the background launched procedure.

Passing parameters to a single well know procedure is easy: the parameters are be added to the message body and the activated procedure looks them up in the received XML, passing them to the called procedure. But is significantly more complex to create a generic mechanism that can pass parameters to any procedure. The problem is the type system, because the parameters have unknown types and the activated procedure has to pass proper typed parameters to the invoked procedure.

A generic solution should accept a variety of parameter types and should deal with the peculiarities of Transact-SQL parameters passing, namely the named parameters capabilities. Also the invocation wrapper <tt>usp_AsyncExecInvoke</tt> should directly accept the parameters for the desired background procedure. After considering sevral alternatives, I settled on the following approach:</>

  * Rely on the generic <a href="http://msdn.microsoft.com/en-us/library/ms173829.aspx" target="_blank">sql_variant</a> data type available in SQL Server. The invocation wrapper <tt>usp_AsyncExecInvoke</tt> accept all the parameters as <tt>sql_variant</tt>.
  * Pass the background procedure parameter names explicitly to the invocation wrapper <tt>usp_AsyncExecInvoke</tt>.
  * Always used named parameters in the background procedure invocation.
  * Build a dynamic SQL batch to invoke the background procedure that deals with the required parameter type conversions.

## Accepting Parameters  


## The invocation wrapper <tt>usp_AsyncExecInvoke<tt> has changed the signature to accept a variable number of parameters:</p> 

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
  The new parameters are used to collect the desired parameter values <i>and names</i> to be passed to the background procedure. The message body will contain the name, type information and value of the parameters. I have added a helper function <tt>fn_DescribeSqlVariant</tt> that uses the <a href="http://msdn.microsoft.com/en-us/library/ms178550.aspx" target="_blank">SQL_VARIANT_PROPERTY</a> system function to extract the type information about the parameters. These properties are then embedded in the message sent to activate the queue procedure:
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">create function </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(<br />
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
</span>
</pre>

<p>
  ...
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">select </span><span style="color:Black">@xmlBody </span><span style="color:Gray">= (<br />
			</span><span style="color:Blue">select </span><span style="color:Black">@procedureName </span><span style="color:Blue">as </span><span style="color:Black">[name]<br />
			</span><span style="color:Gray">, (</span><span style="color:Blue">select </span><span style="color:Gray">* </span><span style="color:Blue">from </span><span style="color:Gray">(<br />
				</span><span style="color:Blue">select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p1</span><span style="color:Gray">, </span><span style="color:Black">@n1</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] </span><span style="color:Blue">WHERE </span><span style="color:Black">@p1 </span><span style="color:Gray">IS NOT NULL<br />
				</span><span style="color:Blue">union all select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p2</span><span style="color:Gray">, </span><span style="color:Black">@n2</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] </span><span style="color:Blue">WHERE </span><span style="color:Black">@p2 </span><span style="color:Gray">IS NOT NULL<br />
				</span><span style="color:Blue">union all select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p3</span><span style="color:Gray">, </span><span style="color:Black">@n3</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] </span><span style="color:Blue">WHERE </span><span style="color:Black">@p3 </span><span style="color:Gray">IS NOT NULL<br />
				</span><span style="color:Blue">union all select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p4</span><span style="color:Gray">, </span><span style="color:Black">@n4</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] </span><span style="color:Blue">WHERE </span><span style="color:Black">@p4 </span><span style="color:Gray">IS NOT NULL<br />
				</span><span style="color:Blue">union all select </span><span style="color:Black">[dbo]</span><span style="color:Gray">.</span><span style="color:Black">[fn_DescribeSqlVariant] </span><span style="color:Gray">(</span><span style="color:Black">@p5</span><span style="color:Gray">, </span><span style="color:Black">@n5</span><span style="color:Gray">) </span><span style="color:Blue">AS </span><span style="color:Black">[*] </span><span style="color:Blue">WHERE </span><span style="color:Black">@p5 </span><span style="color:Gray">IS NOT NULL<br />
				) </span><span style="color:Blue">as </span><span style="color:Black">p </span><span style="color:Blue">for xml path</span><span style="color:Gray">(</span><span style="color:Red">''</span><span style="color:Gray">), </span><span style="color:Blue">type<br />
            </span><span style="color:Gray">) </span><span style="color:Blue">as </span><span style="color:Black">[parameters]<br />
            </span><span style="color:Blue">for xml path</span><span style="color:Gray">(</span><span style="color:Red">'procedure'</span><span style="color:Gray">), </span><span style="color:Blue">type</span><span style="color:Gray">);<br />
        </span><span style="color:Blue">send on conversation </span><span style="color:Black">@h </span><span style="color:Gray">(</span><span style="color:Black">@xmlBody</span><span style="color:Gray">);</span>
</pre>

<p>
  Now to invoke the procedure asynchronously, we would call it like follows:
</p>

<pre>
exec usp_AsyncExecInvoke @procedureName = N'usp_withParam'
 , @p1 = 1.0, @n1 = N'@id'
 , @p2 = N'Foo', @n2='@name'
 , @p3 = 0xBAADF00D, @n3 = '@bytes'
 , @token = @token output;
</pre>

<p>
  The call above corresponds to running the procedure sychronouly like this:
</p>

<h2>
</h2>