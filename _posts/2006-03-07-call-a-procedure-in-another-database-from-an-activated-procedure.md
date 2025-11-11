---
id: 24
title: Call a procedure in another database from an activated procedure
date: 2006-03-07T11:05:58+00:00
author: remus
layout: post
guid: /2006/03/07/call-a-procedure-in-another-database-from-an-activated-procedure/
permalink: /2006/03/07/call-a-procedure-in-another-database-from-an-activated-procedure/
categories:
  - Samples
  - Tutorials
tags:
  - activation
  - Code
  - error
  - sample
  - security
  - service broker
  - signing
---
In my previous post [<font face="Courier New" size="2">/2006/03/01/signing-an-activated-procedure/</font>](/2006/03/01/signing-an-activated-procedure/) <font face="Courier New" size="2">I’ve showed how to use code signing to enable a server level privilege (view server state) when running an activated stored procedure. I’ll show now how to solve a very similar issue: call a stored procedure from another database. Why this is a problem is explained in this post: </font>[<font face="Courier New" size="2">/2006/01/12/why-does-feature-not-work-under-activation/</font>](/2006/01/12/why-does-feature-not-work-under-activation/)

So let’s say the ‘<span>SessionsService’ from my <a href="/2006/03/01/signing-an-activated-procedure/">previous</a> post needs some new functionality: it has to audit the requests it’s receive. An audit infrastructure already exists in the [AuditInfrastructure] database, all is needed is to call the stored procedure [audit_record_request] in that database and the request will be audited. First create this ‘audit infrastructure’, which for our example will be very simple:</p> 

<pre>
<span style="color: Black"></span><span style="color:Blue">create database </span><span style="color:Black">[AuditInfrastructure]<br />
</span><span style="color:Blue">go<br />
 <br />
use </span><span style="color:Black">[AuditInfrastructure]<br />
</span><span style="color:Blue">go<br />
 <br />
</span><span style="color:Green">-- An audit table, simply stores the time of the request<br />
--<br />
</span><span style="color:Blue">create table </span><span style="color:Black">audit_table </span><span style="color:Gray">(</span><span style="color:Black">request_time </span><span style="color:Blue">datetime</span><span style="color:Gray">);<br />
</span><span style="color:Blue">go<br />
 <br />
</span><span style="color:Green">-- this is the audit procedure<br />
--<br />
</span><span style="color:Blue">create procedure </span><span style="color:Black">audit_record_request<br />
</span><span style="color:Blue">as<br />
begin<br />
      set nocount on<br />
      insert into </span><span style="color:Black">audit_table </span><span style="color:Gray">(</span><span style="color:Black">request_time</span><span style="color:Gray">) </span><span style="color:Blue">values </span><span style="color:Gray">(</span><span style="color:Fuchsia">getdate</span><span style="color:Gray">());<br />
</span><span style="color:Blue">end<br />
</span>
</pre>

<p>
  So now all we have to do is to change the SessionService procedure to call this audit procedure:
</p>

<pre><span style="color: Black"></span><span style="color:Blue">USE </span><span style="color:Black">[DemoActivationSigning]</span><span style="color:Gray">;<br />
</span><span style="color:Blue">GO<br />
<br />
ALTER PROCEDURE </span><span style="color:Black">[SessionsServiceProcedure]<br />
</span><span style="color:Blue">AS <br />
BEGIN<br />
      SET NOCOUNT ON</span><span style="color:Gray">;<br />
      </span><span style="color:Blue">DECLARE </span><span style="color:Black">@dh </span><span style="color:Blue">UNIQUEIDENTIFIER</span><span style="color:Gray">;<br />
      </span><span style="color:Blue">DECLARE </span><span style="color:Black">@mt </span><span style="color:Blue">SYSNAME</span><span style="color:Gray">;<br />
 <br />
      </span><span style="color:Blue">BEGIN TRANSACTION</span><span style="color:Gray">;<br />
      </span><span style="color:Blue">WAITFOR </span><span style="color:Gray">(</span><span style="color:Blue">RECEIVE TOP </span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">) </span><span style="color:Black">@dh </span><span style="color:Gray">= </span><span style="color:Blue">conversation_handle</span><span style="color:Gray">,<br />
            </span><span style="color:Black">@mt </span><span style="color:Gray">= </span><span style="color:Black">message_type_name<br />
            </span><span style="color:Blue">FROM </span><span style="color:Black">[SessionsQueue]</span><span style="color:Gray">), </span><span style="color:Blue">TIMEOUT </span><span style="color:Black">1000</span><span style="color:Gray">;<br />
      </span><span style="color:Blue">WHILE </span><span style="color:Gray">(</span><span style="color:Black">@dh </span><span style="color:Gray">IS NOT NULL)<br />
      </span><span style="color:Blue">BEGIN<br />
            </span><span style="color:Black">– </span><span style="color:Blue">If </span><span style="color:Black">the </span><span style="color:Blue">message </span><span style="color:Gray">is </span><span style="color:Black">a request </span><span style="color:Blue">message</span><span style="color:Gray">,<br />
            </span><span style="color:Black">– </span><span style="color:Blue">send </span><span style="color:Black">back a response </span><span style="color:Blue">with </span><span style="color:Black">the list </span><span style="color:Blue">of sessions<br />
            </span><span style="color:Black">–<br />
            </span><span style="color:Blue">IF </span><span style="color:Gray">(</span><span style="color:Black">@mt </span><span style="color:Gray">= </span><span style="color:Black">N‘RequestSessions’</span><span style="color:Gray">)<br />
            </span><span style="color:Blue">BEGIN<br />
                  </span><span style="color:Black">– </span><span style="color:Blue">Get </span><span style="color:Black">the list </span><span style="color:Blue">of current sessions<br />
                  </span><span style="color:Black">– </span><span style="color:Gray">and </span><span style="color:Blue">send </span><span style="color:Black">it back </span><span style="color:Blue">as </span><span style="color:Black">a response<br />
                  –<br />
                  </span><span style="color:Blue">DECLARE </span><span style="color:Black">@response </span><span style="color:Blue">XML</span><span style="color:Gray">;<br />
                  </span><span style="color:Blue">SELECT </span><span style="color:Black">@response </span><span style="color:Gray">= (<br />
                        </span><span style="color:Blue">SELECT </span><span style="color:Gray">* </span><span style="color:Blue">FROM </span><span style="color:Green">sys</span><span style="color:Gray">.</span><span style="color:Green">dm_exec_sessions<br />
                              </span><span style="color:Blue">FOR XML PATH </span><span style="color:Gray">(</span><span style="color:Black">’</span><span style="color:Blue">session</span><span style="color:Black">’</span><span style="color:Gray">), </span><span style="color:Blue">TYPE</span><span style="color:Gray">);<br />
                  </span><span style="color:Blue">SEND ON CONVERSATION </span><span style="color:Black">@dh<br />
                        </span><span style="color:Blue">MESSAGE TYPE </span><span style="color:Black">[Sessions]<br />
                        </span><span style="color:Gray">(</span><span style="color:Black">@response</span><span style="color:Gray">);<br />
                  </span><span style="color:Green">--  This is the extra call to the audit procedure                        <br />
                  </span><span style="color:Blue">EXECUTE </span><span style="color:Black">[AuditInfrastructure]</span><span style="color:Gray">..</span><span style="color:Black">[audit_record_request]</span><span style="color:Gray">; <br />
            </span><span style="color:Blue">END<br />
            </span><span style="color:Black">– </span><span style="color:Blue">End </span><span style="color:Black">the </span><span style="color:Blue">conversation </span><span style="color:Gray">and </span><span style="color:Blue">commit<br />
            END CONVERSATION </span><span style="color:Black">@dh</span><span style="color:Gray">;<br />
            </span><span style="color:Blue">COMMIT</span><span style="color:Gray">;<br />
      <br />
            </span><span style="color:Black">– </span><span style="color:Blue">Try to loop </span><span style="color:Black">once more </span><span style="color:Blue">if </span><span style="color:Black">there are </span><span style="color:Blue">message<br />
            </span><span style="color:Black">–<br />
            </span><span style="color:Blue">SELECT </span><span style="color:Black">@dh </span><span style="color:Gray">= NULL;<br />
            </span><span style="color:Blue">BEGIN TRANSACTION</span><span style="color:Gray">;<br />
            </span><span style="color:Blue">WAITFOR </span><span style="color:Gray">(</span><span style="color:Blue">RECEIVE TOP </span><span style="color:Gray">(</span><span style="color:Black">1</span><span style="color:Gray">) </span><span style="color:Black">@dh </span><span style="color:Gray">= </span><span style="color:Blue">conversation_handle</span><span style="color:Gray">,<br />
                  </span><span style="color:Black">@mt </span><span style="color:Gray">= </span><span style="color:Black">message_type_name<br />
                  </span><span style="color:Blue">FROM </span><span style="color:Black">[SessionsQueue]</span><span style="color:Gray">), </span><span style="color:Blue">TIMEOUT </span><span style="color:Black">1000</span><span style="color:Gray">;<br />
      </span><span style="color:Blue">END<br />
      COMMIT</span><span style="color:Gray">;<br />
</span><span style="color:Blue">END<br />
GO<br /></span>
</pre>

<p>
  So now we send a request to the ‘SessionsService’
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">DECLARE</span><span style="font-size: 10pt; font-family: 'Courier New'"> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DIALOG</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">FROM</span> <span style="color: blue">SERVICE</span> [Requests]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">TO</span> <span style="color: blue">SERVICE</span> <span style="color: red">&#8216;SessionsService&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">ON</span> <span style="color: blue">CONTRACT</span> [SessionsContract]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WITH</span> ENCRYPTION <span style="color: gray">=</span> <span style="color: blue">OFF</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">SEND</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [RequestSessions]<span style="color: gray">;</span><o:p></o:p></span>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <font face="Courier New" size="2">We expect a response back, but nothing happens. The ‘SessionsService’ has gone silent. If we look into the Event Viewer, will find some troublesome entries:</font>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <v:shapetype id="_x0000_t75" stroked="f" filled="f" path="m@4@5l@4@11@9@11@9@5xe" o:preferrelative="t" o:spt="75" coordsize="21600,21600"><v:stroke joinstyle="miter"></v:stroke><v:formulas><v:f eqn="if lineDrawn pixelLineWidth 0"></v:f><v:f eqn="sum @0 1 0"></v:f><v:f eqn="sum 0 0 @1"></v:f><v:f eqn="prod @2 1 2"></v:f><v:f eqn="prod @3 21600 pixelWidth"></v:f><v:f eqn="prod @3 21600 pixelHeight"></v:f><v:f eqn="sum @0 0 1"></v:f><v:f eqn="prod @6 1 2"></v:f><v:f eqn="prod @7 21600 pixelWidth"></v:f><v:f eqn="sum @8 21600 0"></v:f><v:f eqn="prod @7 21600 pixelHeight"></v:f><v:f eqn="sum @10 21600 0"></v:f></v:formulas><v:path o:connecttype="rect" gradientshapeok="t" o:extrusionok="f"></v:path><o:lock aspectratio="t" v:ext="edit"><font style="background-color: #ffffff" color="#008000" face="Courier New">Event Type: Information<br /> Event Source: MSSQLSERVER<br /> Event Category: (2)<br /> Event ID: 9724<br /> Date:  3/7/2006<br /> Time:  10:36:25 AM<br /> User:  REDMOND\remusr<br /> Computer: REMUSR10<br /> Description:<br /> The activated proc [dbo].[SessionsServiceProcedure] running on queue DemoActivationSigning.dbo.SessionsQueue output the following:  &#8216;The server principal &#8220;REDMOND\remusr&#8221; is not able to access the database &#8220;AuditInfrastructure&#8221; under the current security context.&#8217;</font></o:lock></v:shapetype>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <v:shapetype stroked="f" filled="f" path="m@4@5l@4@11@9@11@9@5xe" o:preferrelative="t" o:spt="75" coordsize="21600,21600"><o:lock aspectratio="t" v:ext="edit"><font style="background-color: #ffffff" color="#008000" face="Courier New">For more information, see Help and Support Center at </font><a href="http://go.microsoft.com/fwlink/events.asp"><font style="background-color: #ffffff" color="#008000" face="Courier New">http://go.microsoft.com/fwlink/events.asp</font></a><font style="background-color: #ffffff" color="#008000" face="Courier New">.<br /> Data:<br /> 0000: fc 25 00 00 0a 00 00 00   ü%&#8230;&#8230;<br /> 0008: 09 00 00 00 52 00 45 00   &#8230;.R.E.<br /> 0010: 4d 00 55 00 53 00 52 00   M.U.S.R.<br /> 0018: 31 00 30 00 00 00 14 00   1.0&#8230;..<br /> 0020: 00 00 41 00 75 00 64 00   ..A.u.d.<br /> 0028: 69 00 74 00 49 00 6e 00   i.t.I.n.<br /> 0030: 66 00 72 00 61 00 73 00   f.r.a.s.<br /> 0038: 74 00 72 00 75 00 63 00   t.r.u.c.<br /> 0040: 74 00 75 00 72 00 65 00   t.u.r.e.<br /> 0048: 00 00</font>                     ..<br /> </o:lock></v:shapetype>
</p>

<p class="MsoPlainText" style="margin: 0in 0in 0pt">
  <o:p><font face="Courier New" size="2"> </font></o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  &nbsp;
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 8.5pt; font-family: 'MS Shell Dlg'"><o:p></o:p></span>
</p>

<p>
  <em>Note: In case you wonder ‘so what happened to my request?.The activated stored procedure has thrown an error and it rolled back. Because Service Broker is a fully transactional, the dequeued request was rolled back and is again available for dequeue. The activation will kick in again, causing the same error and again the request to be rolled back. Then again activation will kick in and so on and so forth. Eventually (after 5 consecutive rollbacks) the Service Broker Poisson Message support will detect this situation and will disable the queue.<o:p></o:p></em>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  The problem is, as expected, the database impersonation context, as explained <a href="/2006/01/12/why-does-feature-not-work-under-activation/">here</a>. Same as in the case of server level privileges, the easiest fix is to mark the database trustworthy:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">DATABASE</span> [DemoActivationSigning] <span style="color: blue">SET</span> TRUSTWORTHY <span style="color: blue">ON<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  If we cannot afford this due to security risks (marking the database trustworthy elevates the database dbo to a de-facto sysadmin), we must reside to code signing. The steps are these:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>alter the procedure to have an EXECUTE AS clause (otherwise the code signing infrastructure does not work)
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>create a certificate with a private key in the [DemoActivationSigning] database
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>sign the procedure
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>drop the private key of the certificate
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>copy the certificate into the [AuditInfrastructure] database (backup the certificate to a file and the create from that file)
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>derive a user from the certificate in the [AuditInfrastructure]database
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt 0.5in; text-indent: -0.25in">
  <span>&#8211;<span style="font-family: 'Times New Roman'; font-style: normal; font-variant: normal; font-weight: normal; font-size: 7pt; line-height: normal; font-size-adjust: none; font-stretch: normal">         </span></span>grant the desired privileges to this user
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Here is the code for these steps:
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <o:p> </o:p>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [DemoActivationSigning]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create aprocedure that implements<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; the [SessionsService] service<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">PROCEDURE</span> [SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WITH</span> <span style="color: blue">EXECUTE</span> <span style="color: blue">AS</span> OWNER<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">AS</span><span style="font-size: 10pt; font-family: 'Courier New'"> <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BEGIN<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">SET</span> <span style="color: blue">NOCOUNT</span> <span style="color: blue">ON</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">DECLARE</span> @dh <span style="color: blue">UNIQUEIDENTIFIER</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">DECLARE</span> @mt <span style="color: blue">SYSNAME</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WHILE</span> <span style="color: gray">(</span>@dh <span style="color: gray">IS</span> <span style="color: gray">NOT</span> <span style="color: gray">NULL)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212; If the message is a request message,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212; send back a response with the list of sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">IF</span> <span style="color: gray">(</span>@mt <span style="color: gray">=</span> N<span style="color: red">&#8216;RequestSessions&#8217;</span><span style="color: gray">)<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">BEGIN<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: green">&#8212; Get the list of current sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: green">&#8212; and send it back as a response<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: blue">DECLARE</span> @response <span style="color: blue">XML</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: blue">SELECT</span> @response <span style="color: gray">=</span> <span style="color: gray">(<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                        </span><span style="color: blue">SELECT</span> <span style="color: gray">*</span> <span style="color: blue">FROM</span> <span style="color: green">sys.dm_exec_sessions<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                              </span><span style="color: blue">FOR</span> <span style="color: blue">XML</span> <span style="color: blue">PATH</span> <span style="color: gray">(</span><span style="color: red">&#8216;session&#8217;</span><span style="color: gray">),</span> <span style="color: blue">TYPE</span><span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: blue">SEND</span> <span style="color: blue">ON</span> <span style="color: blue">CONVERSATION</span> @dh<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                        </span><span style="color: blue">MESSAGE</span> <span style="color: blue">TYPE</span> [Sessions]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                        </span><span style="color: gray">(</span>@response<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: blue">EXECUTE</span> [AuditInfrastructure]<span style="color: gray">..</span>[audit_record_request]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212; End the conversation and commit<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">END</span> <span style="color: blue">CONVERSATION</span> @dh<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212; Try to loop once more if there are message<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: green">&#8212;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">SELECT</span> @dh <span style="color: gray">=</span> <span style="color: gray">NULL;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">BEGIN</span> <span style="color: blue">TRANSACTION</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">WAITFOR</span> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: blue">TOP</span> <span style="color: gray">(</span>1<span style="color: gray">)</span> @dh <span style="color: gray">=</span> conversation_handle<span style="color: gray">,<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span>@mt <span style="color: gray">=</span> message_type_name<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>                  </span><span style="color: blue">FROM</span> [SessionsQueue]<span style="color: gray">),</span> TIMEOUT 1000<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">END<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">COMMIT</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">END<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Create a certificate with a private key <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; to sign the procedure with. The password<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; used is not important, we&#8217;ll drop the <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; private key<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span>ENCRYPTION <span style="color: blue">BY</span> PASSWORD <span style="color: gray">=</span> <span style="color: red">&#8216;Password#1234&#8217;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">WITH</span> SUBJECT <span style="color: gray">=</span> <span style="color: red">&#8216;SessionsServiceProcedure Signing<span>  </span>for audit certificate&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Sign the procedure with the certificate&#8217;s private key<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ADD</span><span style="font-size: 10pt; font-family: 'Courier New'"> SIGNATURE <span style="color: blue">TO</span> OBJECT<span style="color: gray">::</span>[SessionsServiceProcedure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">BY</span> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit] <o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>            </span><span style="color: blue">WITH</span> PASSWORD <span style="color: gray">=</span> <span style="color: red">&#8216;Password#1234&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Drop the private key. This way it cannot be<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; used again to sign other procedures.<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span>REMOVE PRIVATE <span style="color: blue">KEY</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Copy the certificate in [master]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; We must backup to a file and create<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; the certificate in [master] from this file<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">BACKUP</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">TO</span> <span style="color: blue">FILE</span> <span style="color: gray">=</span> <span style="color: red">&#8216;C:\SessionsServiceProcedureAudit.CER&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">USE</span><span style="font-size: 10pt; font-family: 'Courier New'"> [AuditInfrastructure]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit]<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><span>      </span><span style="color: blue">FROM</span> <span style="color: blue">FILE</span> <span style="color: gray">=</span> <span style="color: red">&#8216;C:\SessionsServiceProcedureAudit.CER&#8217;</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">CREATE</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: fuchsia">USER</span> [SessionsServiceProcedureAudit] <span style="color: blue">FROM</span> <span style="color: blue">CERTIFICATE</span> [SessionsServiceProcedureAudit]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">G0<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; &#8216;AUTHENTICATE&#8217; permission is required for all other permissions to take effect<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">GRANT</span><span style="font-size: 10pt; font-family: 'Courier New'"> AUTHENTICATE <span style="color: blue">TO</span> [SessionsServiceProcedureAudit]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">GRANT</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">EXECUTE</span> <span style="color: blue">ON</span> [audit_record_request] <span style="color: blue">TO</span> [SessionsServiceProcedureAudit]<span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Enable back the disabled &#8216;SessionsService&#8217; queue<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212;<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">ALTER</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: blue">QUEUE</span> [SessionsQueue] <span style="color: blue">WITH</span> STATUS <span style="color: gray">=</span> <span style="color: blue">ON</span><span style="color: gray">;<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'">GO<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: green; font-family: 'Courier New'">&#8212; Check that the response is now sent back<o:p></o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: blue; font-family: 'Courier New'">WAITFOR</span><span style="font-size: 10pt; font-family: 'Courier New'"> <span style="color: gray">(</span><span style="color: blue">RECEIVE</span> <span style="color: fuchsia">CAST</span><span style="color: gray">(</span>message_body <span style="color: blue">AS</span> <span style="color: blue">XML</span><span style="color: gray">)</span> <span style="color: blue">FROM</span> [RequestsQueue]<span style="color: gray">);<o:p></o:p></span></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  <span style="font-size: 10pt; color: gray; font-family: 'Courier New'"><o:p> </o:p></span>
</p>

<p class="MsoNormal" style="margin: 0in 0in 0pt">
  Notice that I did not sent another request. The existing request was still there, waiting for the procedure to be fixed so it can be processed correctly.
</p>