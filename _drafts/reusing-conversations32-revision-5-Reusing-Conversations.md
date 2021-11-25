---
id: 461
title: Reusing Conversations
date: 2009-07-21T17:37:06+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/07/21/32-revision-5/
permalink: /2009/07/21/32-revision-5/
---
<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana">One of the most common deployed patterns of using Service Broker is what I would call &#8216;data push&#8217;, when Service Broker conversations are used to send data one way only (from initiator to target). In this pattern the target never sends any message back to the initiator. Common applications for this pattern are:<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">ETL from the transactions system to the data warehouse<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">audit and logging<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">aggregation of data from multiple sites<o:p></o:p></span>
</p>

<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana"><o:p> </o:p></span>
</p>

<p style="text-indent: 0in">
  <!--more-->
  
  <span style="font-size: 10pt; color: black; font-family: Verdana">Implementing this pattern is usually very trivial: modify a stored procedure to SEND a message (a datagram) or add a trigger on a table to issue the SEND. But as soon as this is done, the next question that arises is: do I start a new conversation for each message or do I try to reuse conversations? My answer would be that reusing conversations for such patterns (one way data push) is highly recommended.<o:p></o:p></span>
</p>

<p style="text-indent: 0in">
  <span style="font-size: 10pt; color: black; font-family: Verdana">Here are some of my arguments for reusing conversations:<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">conversations are the only elements that <em>are guaranteed</em> to preserve order in transmission (even locally!). E.g. if the order of the events audited is important, they have to be sent on the same conversation<o:p></o:p></span>
</p>

<p style="margin-left: 0.5in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: Symbol"><span>·<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">reusing conversations has a significant performance impact (see <a href="http://rusanu.com/2007/04/03/orlando-slides-and-code/"><font color="#800080">http://rusanu.com/2007/04/03/orlando-slides-and-code/</font></a>)<o:p></o:p></span>
</p>

<p style="margin-left: 1in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: 'Courier New'"><span>o<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">the cost of setting up and tearing down a conversation for each message can influence the performance by ~4x <o:p></o:p></span>
</p>

<p style="margin-left: 1in; text-indent: -0.25in">
  <span style="font-size: 10pt; color: black; font-family: 'Courier New'"><span>o<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal"> </span></span></span><span style="font-size: 10pt; color: black; font-family: Verdana">the advantage on having multiple messages on the same conversation can speed up processing on the RECEIVE side by a factor of ~10x<o:p></o:p></span>
</p>

<ul style="margin-top: 0in" type="disc">
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">conversations are the &#8216;unit of work&#8217; for the Service Broker background and as such resources are allocated in the system according to the number of active conversations (&#8216;active&#8217; = have some activity pending, like sending a message or replying with an acknowledgement). The number of pending messages has less influence on resource allocation (2 dialogs with 500 messages each will allocate far less resources than 1000 dialogs with 1 message each)<o:p></o:p></span>
  </li>
</ul>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">The next step is to decide just how many conversations to use. One conversation would completely preserve order of operations, but because each SEND locks the conversation exclusively, would serialize every transaction and lead to serious scalability problems. My favorite scheme is to have one conversation <em>per <span> </span>user connection</em>. This would preserve the order of actions on one connection, which is most often the desired granularity, and will scale up easily because is very unusual for connections to share transactions (just how often is it that you use <a href="http://msdn2.microsoft.com/en-us/library/ms180061.aspx"><font color="#800080">sp_getbindtoken</font></a>?).<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><span> </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">Next is an example stored proc that reuses conversations based on the conversation arguments (from service, to service, contract) and current connection ID (@@SPID). It&#8217;s based on a table to store the conversation associated with each @@SPID.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<pre><span style="color: Black"></span><span style="color:Green">--&nbsp;This&nbsp;table&nbsp;associates&nbsp;the&nbsp;current&nbsp;connection&nbsp;(@@SPID)<br />
--&nbsp;with&nbsp;a&nbsp;conversation.&nbsp;The&nbsp;association&nbsp;key&nbsp;also<br />
--&nbsp;contains&nbsp;the&nbsp;conversation&nbsp;parameteres:<br />
--&nbsp;from&nbsp;service,&nbsp;to&nbsp;service,&nbsp;contract&nbsp;used<br />
--<br />
</span><span style="color:Blue">CREATE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TABLE</span><span style="color:Black">&nbsp;[SessionConversations]&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">	SPID&nbsp;</span><span style="color:Blue">INT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL,<br />
</span><span style="color:Black">	FromService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL,<br />
</span><span style="color:Black">	ToService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL,<br />
</span><span style="color:Black">	OnContract&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL,<br />
</span><span style="color:Black">	Handle&nbsp;</span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NOT</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL,<br />
</span><span style="color:Black">	</span><span style="color:Blue">PRIMARY</span><span style="color:Black">&nbsp;</span><span style="color:Blue">KEY</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;FromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ToService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;OnContract</span><span style="color:Gray">),<br />
</span><span style="color:Black">	</span><span style="color:Blue">UNIQUE</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">Handle</span><span style="color:Gray">));<br />
</span><span style="color:Black">GO<br />
<br />
</span><span style="color:Green">--&nbsp;SEND&nbsp;procedure.&nbsp;Will&nbsp;lookup&nbsp;to&nbsp;reuse&nbsp;an&nbsp;existing&nbsp;conversation<br />
--&nbsp;or&nbsp;start&nbsp;a&nbsp;new&nbsp;in&nbsp;case&nbsp;no&nbsp;conversation&nbsp;exists&nbsp;or&nbsp;the&nbsp;conversation<br />
--&nbsp;cannot&nbsp;be&nbsp;used<br />
--<br />
</span><span style="color:Blue">CREATE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">PROCEDURE</span><span style="color:Black">&nbsp;[usp_Send]&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">@fromService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@toService&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@onContract&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@messageType&nbsp;</span><span style="color:Blue">SYSNAME</span><span style="color:Gray">,<br />
</span><span style="color:Black">@messageBody&nbsp;</span><span style="color:Blue">VARBINARY</span><span style="color:Gray">(</span><span style="color:Fuchsia">MAX</span><span style="color:Gray">))<br />
</span><span style="color:Blue">AS<br />
BEGIN<br />
</span><span style="color:Black">	</span><span style="color:Blue">SET</span><span style="color:Black">&nbsp;</span><span style="color:Blue">NOCOUNT</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Blue">INT</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">DECLARE</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Blue">INT</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TRANSACTION</span><span style="color:Gray">;<br />
</span><span style="color:Black">	</span><span style="color:Green">--&nbsp;Will&nbsp;need&nbsp;a&nbsp;loop&nbsp;to&nbsp;retry&nbsp;in&nbsp;case&nbsp;the&nbsp;conversation&nbsp;is<br />
</span><span style="color:Black">	</span><span style="color:Green">--&nbsp;in&nbsp;a&nbsp;state&nbsp;that&nbsp;does&nbsp;not&nbsp;allow&nbsp;transmission<br />
</span><span style="color:Black">	</span><span style="color:Green">--<br />
</span><span style="color:Black">	</span><span style="color:Blue">WHILE</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">=</span><span style="color:Black">1</span><span style="color:Gray">)<br />
</span><span style="color:Black">	</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Seek&nbsp;an&nbsp;eligible&nbsp;conversation&nbsp;in&nbsp;[SessionConversations]<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;Handle<br />
			</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[SessionConversations]<br />
			</span><span style="color:Blue">WHERE</span><span style="color:Black">&nbsp;SPID&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@SPID<br />
</span><span style="color:Black">			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;FromService&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@fromService<br />
			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;ToService&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@toService<br />
			</span><span style="color:Gray">AND</span><span style="color:Black">&nbsp;OnContract&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@OnContract</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">IS</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL<br />
</span><span style="color:Black">		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Need&nbsp;to&nbsp;start&nbsp;a&nbsp;new&nbsp;conversation&nbsp;for&nbsp;the&nbsp;current&nbsp;@@spid<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">BEGIN</span><span style="color:Black">&nbsp;</span><span style="color:Blue">DIALOG</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle<br />
				</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;</span><span style="color:Blue">SERVICE</span><span style="color:Black">&nbsp;@fromService<br />
				</span><span style="color:Blue">TO</span><span style="color:Black">&nbsp;</span><span style="color:Blue">SERVICE</span><span style="color:Black">&nbsp;@toService<br />
				</span><span style="color:Blue">ON</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONTRACT</span><span style="color:Black">&nbsp;@onContract<br />
				</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ENCRYPTION</span><span style="color:Black">&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Blue">OFF</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">INSERT</span><span style="color:Black">&nbsp;</span><span style="color:Blue">INTO</span><span style="color:Black">&nbsp;[SessionConversations]<br />
				</span><span style="color:Gray">(</span><span style="color:Black">SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;FromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;ToService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;OnContract</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;Handle</span><span style="color:Gray">)<br />
</span><span style="color:Black">				</span><span style="color:Blue">VALUES<br />
</span><span style="color:Black">				</span><span style="color:Gray">(</span><span style="color:Fuchsia">@@SPID</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@fromService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@toService</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@onContract</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">);<br />
</span><span style="color:Black">		</span><span style="color:Blue">END</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Attempt&nbsp;to&nbsp;SEND&nbsp;on&nbsp;the&nbsp;associated&nbsp;conversation<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">SEND</span><span style="color:Black">&nbsp;</span><span style="color:Blue">ON</span><span style="color:Black">&nbsp;</span><span style="color:Blue">CONVERSATION</span><span style="color:Black">&nbsp;@handle<br />
			</span><span style="color:Blue">MESSAGE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">TYPE</span><span style="color:Black">&nbsp;@messageType<br />
			</span><span style="color:Gray">(</span><span style="color:Black">@messageBody</span><span style="color:Gray">);<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">@@ERROR</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@error&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;0<br />
		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;Successful&nbsp;send,&nbsp;just&nbsp;exit&nbsp;the&nbsp;loop<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">BREAK</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@counter</span><span style="color:Gray">+</span><span style="color:Black">1</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">IF</span><span style="color:Black">&nbsp;@counter&nbsp;</span><span style="color:Gray">&gt;</span><span style="color:Black">&nbsp;10<br />
		</span><span style="color:Blue">BEGIN<br />
</span><span style="color:Black">			</span><span style="color:Green">--&nbsp;We&nbsp;failed&nbsp;10&nbsp;times&nbsp;in&nbsp;a&nbsp;row,&nbsp;something&nbsp;must&nbsp;be&nbsp;broken<br />
</span><span style="color:Black">			</span><span style="color:Green">--<br />
</span><span style="color:Black">			</span><span style="color:Blue">RAISERROR</span><span style="color:Black">&nbsp;</span><span style="color:Gray">(<br />
</span><span style="color:Black">				N</span><span style="color:Red">'Failed&nbsp;to&nbsp;SEND&nbsp;on&nbsp;a&nbsp;conversation&nbsp;for&nbsp;more&nbsp;than&nbsp;10&nbsp;times.&nbsp;Error&nbsp;%i.'<br />
</span><span style="color:Black">				</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;16</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;1</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;@error</span><span style="color:Gray">)</span><span style="color:Black">&nbsp;</span><span style="color:Blue">WITH</span><span style="color:Black">&nbsp;</span><span style="color:Fuchsia">LOG</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">BREAK</span><span style="color:Gray">;<br />
</span><span style="color:Black">		</span><span style="color:Blue">END<br />
</span><span style="color:Black">		</span><span style="color:Green">--&nbsp;Delete&nbsp;the&nbsp;associated&nbsp;conversation&nbsp;from&nbsp;the&nbsp;table&nbsp;and&nbsp;try&nbsp;again<br />
</span><span style="color:Black">		</span><span style="color:Green">--<br />
</span><span style="color:Black">		</span><span style="color:Blue">DELETE</span><span style="color:Black">&nbsp;</span><span style="color:Blue">FROM</span><span style="color:Black">&nbsp;[SessionConversations]<br />
			</span><span style="color:Blue">WHERE</span><span style="color:Black">&nbsp;Handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;@handle</span><span style="color:Gray">;<br />
</span><span style="color:Black">			</span><span style="color:Blue">SELECT</span><span style="color:Black">&nbsp;@handle&nbsp;</span><span style="color:Gray">=</span><span style="color:Black">&nbsp;</span><span style="color:Gray">NULL;<br />
</span><span style="color:Black">	</span><span style="color:Blue">END<br />
</span><span style="color:Black">	</span><span style="color:Blue">COMMIT</span><span style="color:Gray">;<br />
</span><span style="color:Blue">END<br />
</span><span style="color:Black">GO<br />
<br />
</span><span style="color:Green">--&nbsp;Test&nbsp;the&nbsp;procedure&nbsp;with&nbsp;some&nbsp;dummy&nbsp;paramaters.&nbsp;<br />
--&nbsp;The&nbsp;FromService&nbsp;‘ST’&nbsp;must&nbsp;already&nbsp;exist&nbsp;in&nbsp;the&nbsp;database<br />
--<br />
</span><span style="color:Blue">exec</span><span style="color:Black">&nbsp;usp_Send&nbsp;N</span><span style="color:Red">'ST'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'Foobar'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'DEFAULT'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;N</span><span style="color:Red">'DEFAULT'</span><span style="color:Gray">,</span><span style="color:Black">&nbsp;0x01020304</span><span style="color:Gray">;</span>
</pre>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">This shows how to reuse the conversations, but there are still plenty of details one has to think of, like:<o:p></o:p></span>
</p>

<ul style="margin-top: 0in" type="disc">
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">how long should a conversation be used for?<o:p></o:p></span>
  </li>
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">who should end these reused conversations?<o:p></o:p></span>
  </li>
  <li class="MsoNormal" style="margin: 0in 0in 0pt">
    <span style="font-size: 10pt; font-family: Verdana">how to treat conversation errors?<o:p></o:p></span>
  </li>
</ul>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: Verdana">I&#8217;ll try to visit these topics in a future post.</span>
</p>

<img src="http://blogs.msdn.com/aggbug.aspx?PostID=2265845" height="1" width="1" />