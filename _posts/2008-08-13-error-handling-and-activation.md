---
id: 103
title: Error Handling and Activation
date: 2008-08-13T18:44:59+00:00
author: remus
layout: post
guid: /?p=103
permalink: /2008/08/13/error-handling-and-activation/
categories:
  - Troubleshooting
  - Tutorials
tags:
  - activation
  - error
  - service broker
---
I have previously talked <a href="/2008/08/03/understanding-queue-monitors/" target="_blank">here</a> about the queue monitors and the role they play in launching activated procedures. If you recall, I&#8217;ve mentioned that the Queue Monitors will enter the <span style="color:Green">NOTIFIED</span> state after they launch a stored procedure and not launch again the procedure until the Queue Monitor notices that <span style="color:Blue">RECEIVE</span> statements are being issued against the queue. In an older <a href="/2007/10/31/error-handling-in-service-broker-procedures/" target="_blank">post</a> I have also talked about how difficult is to get error handling right, and in particular cast and convention errors. This may seem a trivial problem but in the Service Broker programs it is actually a serious problem because of the frequent conversation to and from XML. These two separate issues can actually &#8216;cooperate&#8217; into a somehow surprising behavior. Namely if the activated procedure hits an error _before_ it completes the <span style="color:Blue">RECEIVE</span> statement, the Queue Monitor will stay in <span style="color:Green">NOTIFIED</span> state and won&#8217;t activate again the procedure. Although this looks similar to the typical poison message behavior when the queue is automatically disabled, this is a different issue. And unlike the poison message case, in the case when the Queue Monitor is stranded in <span style="color:Green">NOTIFIED</span> state you can **not** get a notification that the queue is no longer functional.

This post continues with an example showing how a relatively safe activated procedure can end up in this situation.

<!--more-->

Lets consider what happens when a typical Service Broker activated procedure is presented with an invalid XML fragment in a message. First, lets create a pair of services:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> [initiator]</span><span style="color:Gray">;
</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> [/2008/08/13/initiator] </span><span style="color:Blue">on</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> [initiator]</span><span style="color:Gray">;
</span><span style="color:Black">go

</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> [target]</span><span style="color:Gray">;
</span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> [/2008/08/13/target]
	</span><span style="color:Blue">on</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> [target]
	</span><span style="color:Gray">(</span><span style="color:Black">[DEFAULT]</span><span style="color:Gray">);
</span><span style="color:Black">go</span></pre>

Lets say the target service is used to log user logins from workstations. We have a table that keeps these login events:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">table</span><span style="color:Black"> [user_log] </span><span style="color:Gray">(
</span><span style="color:Black">	[id] </span><span style="color:Blue">int</span><span style="color:Black"> </span><span style="color:Gray">not</span><span style="color:Black"> </span><span style="color:Gray">null</span><span style="color:Black"> </span><span style="color:Blue">identity</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">,</span><span style="color:Black">1</span><span style="color:Gray">)</span><span style="color:Black"> </span><span style="color:Blue">primary</span><span style="color:Black"> </span><span style="color:Blue">key</span><span style="color:Gray">,
</span><span style="color:Black">	[user_name] </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">256</span><span style="color:Gray">),
</span><span style="color:Black">	[workstation] </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">256</span><span style="color:Gray">),
</span><span style="color:Black">	[login_time] </span><span style="color:Blue">datetime</span><span style="color:Black"> </span><span style="color:Gray">not</span><span style="color:Black"> </span><span style="color:Gray">null);
</span><span style="color:Black">go</span></pre>

The initiator service is sending XML messages that embed the user, workstation and time for a login event:

<pre><span style="color:Gray">&lt;login_info xmlns="//rusanu.com/2008/08/13" time="2008-08-13T08:00:00"&gt;
  &lt;user&gt;John Doe&lt;/user&gt;
  &lt;workstation&gt;PCLAB36X&lt;/workstation&gt;
&lt;/login_info&gt;
</span></pre>

So we create an activated procedure that is extracting the name, workstation and time from the XML and inserts the information into the [user_log] table:

<pre><span style="color: Black"></span><span style="color:Blue">create</span><span style="color:Black"> </span><span style="color:Blue">procedure</span><span style="color:Black"> usp_target
</span><span style="color:Blue">as
begin
</span><span style="color:Black">	</span><span style="color:Blue">set</span><span style="color:Black"> </span><span style="color:Blue">nocount</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @h </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @mt </span><span style="color:Blue">sysname</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @msg </span><span style="color:Blue">xml</span><span style="color:Gray">;

</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">try
</span><span style="color:Black">		</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">transaction
</span><span style="color:Black">
		</span><span style="color:Blue">waitfor</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">receive</span><span style="color:Black"> </span><span style="color:Blue">top</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">)</span><span style="color:Black">
			@h </span><span style="color:Gray">=</span><span style="color:Black"> [conversation_handle]</span><span style="color:Gray">,
</span><span style="color:Black">			@mt </span><span style="color:Gray">=</span><span style="color:Black"> [message_type_name]</span><span style="color:Gray">,
</span><span style="color:Black">			@msg </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">message_body </span><span style="color:Blue">as</span><span style="color:Black"> </span><span style="color:Blue">xml</span><span style="color:Gray">)
</span><span style="color:Black">			</span><span style="color:Blue">from</span><span style="color:Black"> [target]</span><span style="color:Gray">),</span><span style="color:Black"> </span><span style="color:Blue">timeout</span><span style="color:Black"> 1000</span><span style="color:Gray">;
</span><span style="color:Black">
		</span><span style="color:Blue">if</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Black">@h </span><span style="color:Gray">is</span><span style="color:Black"> </span><span style="color:Gray">not</span><span style="color:Black"> </span><span style="color:Gray">null)
</span><span style="color:Black">		</span><span style="color:Blue">begin
</span><span style="color:Black">			</span><span style="color:Blue">if</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Black">@mt </span><span style="color:Gray">=</span><span style="color:Black"> N</span><span style="color:Red">'DEFAULT'</span><span style="color:Gray">)
</span><span style="color:Black">			</span><span style="color:Blue">begin
</span><span style="color:Black">				</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">xmlnamespaces</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">DEFAULT</span><span style="color:Black"> </span><span style="color:Red">'//rusanu.com/2008/08/13'</span><span style="color:Gray">)
</span><span style="color:Black">				</span><span style="color:Blue">insert</span><span style="color:Black"> </span><span style="color:Blue">into</span><span style="color:Black"> [user_log] </span><span style="color:Gray">(
</span><span style="color:Black">					[user_name]</span><span style="color:Gray">,
</span><span style="color:Black">					[workstation]</span><span style="color:Gray">,
</span><span style="color:Black">					[login_time]</span><span style="color:Gray">)
</span><span style="color:Black">				</span><span style="color:Blue">select</span><span style="color:Black"> T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'user[1]'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'varchar(256)'</span><span style="color:Gray">),</span><span style="color:Black">
					T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'workstation[1]'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'varchar(256)'</span><span style="color:Gray">),
</span><span style="color:Black">					T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'@time'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'datetime'</span><span style="color:Gray">)
</span><span style="color:Black">					</span><span style="color:Blue">from</span><span style="color:Black"> @msg</span><span style="color:Gray">.</span><span style="color:Black">nodes</span><span style="color:Gray">(</span><span style="color:Red">'/login_info'</span><span style="color:Gray">)</span><span style="color:Black"> T</span><span style="color:Gray">(</span><span style="color:Black">c</span><span style="color:Gray">);
</span><span style="color:Black">			</span><span style="color:Blue">end
</span><span style="color:Black">
			</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h</span><span style="color:Gray">;
</span><span style="color:Black">		</span><span style="color:Blue">end</span><span style="color:Black">
		</span><span style="color:Blue">commit</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">try
</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">catch
</span><span style="color:Black">		</span><span style="color:Blue">declare</span><span style="color:Black"> @error </span><span style="color:Blue">int</span><span style="color:Gray">,</span><span style="color:Black"> @message </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">4000</span><span style="color:Gray">),</span><span style="color:Black"> @xstate </span><span style="color:Blue">int</span><span style="color:Gray">;
</span><span style="color:Black">		</span><span style="color:Blue">select</span><span style="color:Black"> @error </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">(),</span><span style="color:Black"> @message </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">(),</span><span style="color:Black"> @xstate </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black"> @xstate </span><span style="color:Gray">!=</span><span style="color:Black"> 0
			</span><span style="color:Blue">rollback</span><span style="color:Gray">;
</span><span style="color:Black">
		</span><span style="color:Green">----------------------------
</span><span style="color:Black">		</span><span style="color:Green">-- Error logging goes here
</span><span style="color:Black">		</span><span style="color:Green">----------------------------
</span><span style="color:Black">		</span><span style="color:Blue">print</span><span style="color:Black"> @message</span><span style="color:Gray">;
</span><span style="color:Black">
	</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">catch</span><span style="color:Black">
</span><span style="color:Blue">end
</span><span style="color:Black">go</span>
<br />
<span style="color:Blue">alter</span><span style="color:Black"> </span><span style="color:Blue">queue</span><span style="color:Black"> [target] </span><span style="color:Blue">with</span><span style="color:Black"> activation </span><span style="color:Gray">(
</span><span style="color:Black">	</span><span style="color:Blue">status</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Gray">,
</span><span style="color:Black">	procedure_name </span><span style="color:Gray">=</span><span style="color:Black"> [usp_target]</span><span style="color:Gray">,
</span><span style="color:Black">	</span><span style="color:Blue">execute</span><span style="color:Black"> </span><span style="color:Blue">as</span><span style="color:Black"> </span><span style="color:Blue">owner</span><span style="color:Gray">,
</span><span style="color:Black">	max_queue_readers </span><span style="color:Gray">=</span><span style="color:Black"> 1</span><span style="color:Gray">);
</span><span style="color:Black">go</span></pre>

So we go ahead and test the procedure, by sending a message that will activate the procedure:

<pre><span style="color:Blue">declare</span><span style="color:Black"> @h </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;
</span><span style="color:Blue">declare</span><span style="color:Black"> @msg </span><span style="color:Blue">nvarchar</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">);
</span><span style="color:Blue">select</span><span style="color:Black"> @msg </span><span style="color:Gray">=</span><span style="color:Black"> N</span><span style="color:Red">'&lt;?xml version="1.0" encoding="utf-8"?&gt;
&lt;login_info xmlns="//rusanu.com/2008/08/13" time="2008-08-13T08:00:00.000"&gt;
  &lt;user&gt;John Doe&lt;/user&gt;
  &lt;workstation&gt;PCLAB36X&lt;/workstation&gt;
&lt;/login_info&gt;'</span><span style="color:Gray">;
</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">dialog</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h
	</span><span style="color:Blue">from</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> [/2008/08/13/initiator]
	</span><span style="color:Blue">to</span><span style="color:Black"> </span><span style="color:Blue">service</span><span style="color:Black"> N</span><span style="color:Red">'/2008/08/13/target'
</span><span style="color:Black">	</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">encryption</span><span style="color:Black"> </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Blue">off</span><span style="color:Gray">;
</span><span style="color:Blue">send</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h </span><span style="color:Gray">(</span><span style="color:Black">@msg</span><span style="color:Gray">);
</span><span style="color:Black">go</span></pre>

And when we check the [user_log] table, nothing is there. How come? Well, we did send an invalid XML message. Because we assigned the XML to an NVARCHAR variable, it is stored as Unicode encoding. But our XML fragment declares itself as UTF-8 encoding, hence it is invalid and casting it to XML datatype will fail. This sort of XML errors are less obvious and I&#8217;ve seen in real life situations encoding mixed up like this.

If we check the server ERRORLOG or the system Event Log (eventvwr.exe) we&#8217;ll find the error message from the XML convertion:

<tt>The activated proc [dbo].[usp_target] running on queue test.dbo.target output the following: 'XML parsing: line 1, character 38, unable to switch the encoding'</tt>

This error happens _during_ the <span style="color:Blue">RECEIVE</span> statement. This interrupts the statement _before_ the Queue Monitor is notified that RECEIVEs are occurring, and thus the activated procedure exists but the Queue Monitor is left in <span style="color:Green">NOTIFIED</span> state. We can check this in the DMV <span style="color:Green">sys.dm_broker_queue_monitors</span>. Although the queue is active, it refuses to activate again the procedure.

The workaround for this is failry easy: simply move the cast **after** the <span style="color:Blue">RECEIVE</span> statement:

<pre><span style="color:Blue">alter</span><span style="color:Black"> </span><span style="color:Blue">procedure</span><span style="color:Black"> usp_target
</span><span style="color:Blue">as
begin
</span><span style="color:Black">	</span><span style="color:Blue">set</span><span style="color:Black"> </span><span style="color:Blue">nocount</span><span style="color:Black"> </span><span style="color:Blue">on</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @h </span><span style="color:Blue">uniqueidentifier</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @mt </span><span style="color:Blue">sysname</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @msg </span><span style="color:Blue">xml</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">declare</span><span style="color:Black"> @msg_bin </span><span style="color:Blue">varbinary</span><span style="color:Gray">(</span><span style="color:Fuchsia">max</span><span style="color:Gray">);

</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">try
</span><span style="color:Black">		</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">transaction
</span><span style="color:Black">
		</span><span style="color:Blue">waitfor</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">receive</span><span style="color:Black"> </span><span style="color:Blue">top</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">)</span><span style="color:Black">
			@h </span><span style="color:Gray">=</span><span style="color:Black"> [conversation_handle]</span><span style="color:Gray">,
</span><span style="color:Black">			@mt </span><span style="color:Gray">=</span><span style="color:Black"> [message_type_name]</span><span style="color:Gray">,
</span><span style="color:Black">			@msg_bin </span><span style="color:Gray">=</span><span style="color:Black"> [message_body]
			</span><span style="color:Blue">from</span><span style="color:Black"> [target]</span><span style="color:Gray">),</span><span style="color:Black"> </span><span style="color:Blue">timeout</span><span style="color:Black"> 1000</span><span style="color:Gray">;
</span><span style="color:Black">
		</span><span style="color:Blue">if</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Black">@h </span><span style="color:Gray">is</span><span style="color:Black"> </span><span style="color:Gray">not</span><span style="color:Black"> </span><span style="color:Gray">null)
</span><span style="color:Black">		</span><span style="color:Blue">begin
</span><span style="color:Black">			</span><span style="color:Blue">if</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Black">@mt </span><span style="color:Gray">=</span><span style="color:Black"> N</span><span style="color:Red">'DEFAULT'</span><span style="color:Gray">)
</span><span style="color:Black">			</span><span style="color:Blue">begin
</span><span style="color:Black">				</span><span style="color:Blue">select</span><span style="color:Black"> @msg </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">cast</span><span style="color:Gray">(</span><span style="color:Black">@msg_bin </span><span style="color:Blue">as</span><span style="color:Black"> </span><span style="color:Blue">xml</span><span style="color:Gray">);
</span><span style="color:Black">				</span><span style="color:Blue">with</span><span style="color:Black"> </span><span style="color:Blue">xmlnamespaces</span><span style="color:Black"> </span><span style="color:Gray">(</span><span style="color:Blue">DEFAULT</span><span style="color:Black"> </span><span style="color:Red">'//rusanu.com/2008/08/13'</span><span style="color:Gray">)
</span><span style="color:Black">				</span><span style="color:Blue">insert</span><span style="color:Black"> </span><span style="color:Blue">into</span><span style="color:Black"> [user_log] </span><span style="color:Gray">(
</span><span style="color:Black">					[user_name]</span><span style="color:Gray">,
</span><span style="color:Black">					[workstation]</span><span style="color:Gray">,
</span><span style="color:Black">					[login_time]</span><span style="color:Gray">)
</span><span style="color:Black">				</span><span style="color:Blue">select</span><span style="color:Black"> T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'user[1]'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'varchar(256)'</span><span style="color:Gray">),</span><span style="color:Black">
					T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'workstation[1]'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'varchar(256)'</span><span style="color:Gray">),
</span><span style="color:Black">					T</span><span style="color:Gray">.</span><span style="color:Black">c</span><span style="color:Gray">.</span><span style="color:Black">value</span><span style="color:Gray">(</span><span style="color:Red">'@time'</span><span style="color:Gray">,</span><span style="color:Black"> </span><span style="color:Red">'datetime'</span><span style="color:Gray">)
</span><span style="color:Black">					</span><span style="color:Blue">from</span><span style="color:Black"> @msg</span><span style="color:Gray">.</span><span style="color:Black">nodes</span><span style="color:Gray">(</span><span style="color:Red">'/login_info'</span><span style="color:Gray">)</span><span style="color:Black"> T</span><span style="color:Gray">(</span><span style="color:Black">c</span><span style="color:Gray">);
</span><span style="color:Black">			</span><span style="color:Blue">end
</span><span style="color:Black">
			</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">conversation</span><span style="color:Black"> @h</span><span style="color:Gray">;
</span><span style="color:Black">		</span><span style="color:Blue">end</span><span style="color:Black">
		</span><span style="color:Blue">commit</span><span style="color:Gray">;
</span><span style="color:Black">	</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">try
</span><span style="color:Black">	</span><span style="color:Blue">begin</span><span style="color:Black"> </span><span style="color:Blue">catch
</span><span style="color:Black">		</span><span style="color:Blue">declare</span><span style="color:Black"> @error </span><span style="color:Blue">int</span><span style="color:Gray">,</span><span style="color:Black"> @message </span><span style="color:Blue">varchar</span><span style="color:Gray">(</span><span style="color:Black">4000</span><span style="color:Gray">),</span><span style="color:Black"> @xstate </span><span style="color:Blue">int</span><span style="color:Gray">;
</span><span style="color:Black">		</span><span style="color:Blue">select</span><span style="color:Black"> @error </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">ERROR_NUMBER</span><span style="color:Gray">(),</span><span style="color:Black"> @message </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">ERROR_MESSAGE</span><span style="color:Gray">(),</span><span style="color:Black"> @xstate </span><span style="color:Gray">=</span><span style="color:Black"> </span><span style="color:Fuchsia">XACT_STATE</span><span style="color:Gray">();
</span><span style="color:Black">		</span><span style="color:Blue">if</span><span style="color:Black"> @xstate </span><span style="color:Gray">!=</span><span style="color:Black"> 0
			</span><span style="color:Blue">rollback</span><span style="color:Gray">;
</span><span style="color:Black">
		</span><span style="color:Green">----------------------------
</span><span style="color:Black">		</span><span style="color:Green">-- Error logging goes here
</span><span style="color:Black">		</span><span style="color:Green">----------------------------
</span><span style="color:Black">		</span><span style="color:Blue">print</span><span style="color:Black"> @message</span><span style="color:Gray">;
</span><span style="color:Black">
	</span><span style="color:Blue">end</span><span style="color:Black"> </span><span style="color:Blue">catch</span><span style="color:Black">
</span><span style="color:Blue">end
</span><span style="color:Black">go
</span></pre>

With this modified procedure the conversation error is causing the <span style="color:Blue">RECEIVE</span> to rollback, but the Queue Monitor has already entered the <span style="color:Green">RECEIVES_OCCURRING</span> state and will activate the procedure again. Five consecutive rollbacks are triggering the poison message detection mechanism and the queue is disabled as expected. Why is this a better behavior than before? Because you can have monitoring set up around the BROKER\_QUEUE\_DISABLED event notification and intervene: remove the poison message and re-activate the queue.

### XML message validation

In my example I intentionally used the DEFAULT message type, with no validation, and did not use a message type with well\_formed\_xml message validation. The later would had muted the whole point of the errors occurring during XML cast since and invalid message would not event be enqueued int the [target] queue. Indeed, using message type validation is the best option for such a case, but none the less, I have seen situations like the one I&#8217;ve described in production. It can happen when the developers intentionally omit the XML message type validation in the message type declaration to gain a bit of performance, or because they consider themselves safe from an invalid XML ever being sent.

## Conclusion

Always make sure your <span style="color:Blue">RECEIVE</span> statement is being executed in your activated procedure. If you are doing any cast, is better to receive into intermediate types that do not require a cast and only perform the cast _after_ the <span style="color:Blue">RECEIVE</span> statement has finished. Otherwise you may end up with your service not activating but your notification infrastructure never detecting this problem.