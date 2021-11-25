---
id: 436
title: Exception handling and nested transactions
date: 2009-06-11T10:07:20+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/06/11/430-revision-6/
permalink: /2009/06/11/430-revision-6/
---
I wanted to use a template for writing procedures that behave as intuitively as possible in regard to nested transactions. My goals were:

<ul style="list-style-type:disc">
  <li>
    The procedure template should wrap all the work done in the procedure in a transaction.
  </li>
  <li>
    The procedures should be able to call each other and the calee should nest its transaction inside the outer caller transaction.
  </li>
  <li>
    The procedure should only rollback its own work in case of exception, if possible
  </li>
  <li>
    The caller should be able to resume and continue even if the calee rolled back ts work.
  </li>
</ul>

My current template I&#8217;ve been using for some time and so far I did not encounter problems with it. The procedures start a new transaction if no transaction is pending. Otherwise they simply create a savepoint. On exit the procedure commits the transaction they started (if they started one), otherwise they simply exit. On exception, if the transaction is not doomed, the procedure either rolls back (if it started the transaction), or rolls back to the savepoint it created (if calee already provided a transaction).

Because of the use of savepoints this template does not work in all situations, since there are cases like distributed transactions, that cannot be mixed with savepoints. But for the average run of the mill procedures, this template has saved me very well.

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black">&nbsp;</span><span style="color:Blue">procedure</span><span style="color:Black">&nbsp;[usp_my_procedure_name]<br />
</span><span style="color:Blue">as<br />
begin<br />
</span><span style="color:Black">	</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;</span><span style="color:Blue">nocount</span><span style="color:Black">&nbsp;</span><span style="color:Blue">on</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Blue">int</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">set</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@trancount</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">try<br />
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
			</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction<br />
</span><span style="color:Black">		</span><span style="color:Blue">else<br />
</span><span style="color:Black">			</span><span style="color:Blue">save</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction</span><span style="color:Black">&nbsp;usp_my_procedure_name</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Do&nbsp;the&nbsp;actual&nbsp;work&nbsp;here<br />
</span><span style="color:Black">	<br />
lbexit</span><span style="color:Gray">:<br />
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0	<br />
			</span><span style="color:Blue">commit</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">end</span><span style="color:Black">&nbsp;</span><span style="color:Blue">try<br />
</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black">&nbsp;</span><span style="color:Blue">catch<br />
</span><span style="color:Black">		</span><span style="color:Blue">declare</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Blue">int</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@message&nbsp;</span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">4000</span><span style="color:Gray">),</span><span style="color:Black">&nbsp;@xstate&nbsp;</span><span style="color:Blue">int</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">select</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">(),</span><span style="color:Black">&nbsp;@message&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">(),</span><span style="color:Black">&nbsp;@xstate&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();<br />
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xstate&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Gray">-</span><span style="color:Black">1<br />
			</span><span style="color:Blue">rollback</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xstate&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1&nbsp;</span><span style="color:Gray">and</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
			</span><span style="color:Blue">rollback<br />
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black">&nbsp;@xstate&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1&nbsp;</span><span style="color:Gray">and</span><span style="color:Black">&nbsp;@trancount&nbsp;</span><span style="color:Gray">&gt;</span><span style="color:Black">&nbsp;0<br />
			</span><span style="color:Blue">rollback</span><span style="color:Black">&nbsp;</span><span style="color:Blue">transaction</span><span style="color:Black">&nbsp;usp_my_procedure_name</span><span style="color:Gray">;<br />
<br />
</span><span style="color:Black">		</span><span style="color:Blue">raiserror</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Red">'usp_my_procedure_name:&nbsp;%d:&nbsp;%s'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;11</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@error</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@message</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">return</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">end</span><span style="color:Black">&nbsp;</span><span style="color:Blue">catch</span><span style="color:Black">	<br />
</span><span style="color:Blue">end</span>
</pre>