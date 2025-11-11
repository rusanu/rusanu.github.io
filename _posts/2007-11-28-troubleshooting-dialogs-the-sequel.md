---
id: 65
title: Troubleshooting dialogs, the sequel
date: 2007-11-28T19:00:47+00:00
author: remus
layout: post
guid: /2007/11/28/troubleshooting-dialogs-the-sequel/
permalink: /2007/11/28/troubleshooting-dialogs-the-sequel/
categories:
  - Announcements
  - Troubleshooting
---
<font face="Times New Roman">Almost two years ago I have posted the original </font>[<font color="#800080" face="Times New Roman">Troubleshooting dialogs</font>](/2005/12/20/troubleshooting-dialogs/) <font face="Times New Roman">post in this blog (back in the day when it was hosted at MSDN). I have often referenced this post when I was asked to help investigate some Service Broker issue, and I have seen others referencing it. Although it lacked a lot of details it was a good starting point for anybody who was asking itself &#8216;The messages don&#8217;t get through, wonder what&#8217;s wrong&#8230;&#8217;</font>

<font face="Times New Roman">It is time for an updated entry on this topic because the bar was raised to a whole new level: the Service Broker Diagnostics Tool was shipped in the November CTP of SQL Server 2008! <!--more--><v :shapetype coordsize="21600,21600" o:spt="75" o:preferrelative="t" path="m@4@5l@4@11@9@11@9@5xe" filled="f" stroked="f" id="_x0000_t75"></v><v :stroke joinstyle="miter"></v><v :formulas></v><v :f eqn="if lineDrawn pixelLineWidth 0"></v><v :f eqn="sum @0 1 0"></v><v :f eqn="sum 0 0 @1"></v><v :f eqn="prod @2 1 2"></v><v :f eqn="prod @3 21600 pixelWidth"></v><v :f eqn="prod @3 21600 pixelHeight"></v><v :f eqn="sum @0 0 1"></v><v :f eqn="prod @6 1 2"></v><v :f eqn="prod @7 21600 pixelWidth"></v><v :f eqn="sum @8 21600 0"></v><v :f eqn="prod @7 21600 pixelHeight"></v><v :f eqn="sum @10 21600 0"></v><v :path o:extrusionok="f" gradientshapeok="t" o:connecttype="rect"></v><o :lock v:ext="edit" aspectratio="t"></o></font><v :shape type="#\_x0000\_t75" alt="More..." style="width: 49.5pt; height: 7.5pt" id="\_x0000\_i1025"></v><v :imagedata src="file:///C:\DOCUME~1\Remus\LOCALS~1\Temp\msohtml1\01\clip_image001.gif" o:href="/wp-includes/js/tinymce/themes/advanced/images/spacer.gif"></v>

<font face="Times New Roman">So what is this new Diagnostics Tool? It is a command line utility that will investigate the configuration of a service deployment and will check if all the necessary requirement to exchange messages with another service are met. The tool basically contains all the know-how on how to troubleshoot Service Broker issues and has the means to collect configuration data, analyze the configuration and offer a diagnostic on the spot. Although everything the tool does could also be done by an experienced dba by looking at the same configuration data the tool is looking at, the fact that tool does it automatically makes it much easier. And besides, not everybody is an expert dba and even less an expert Service Broker configuration dba. Note that although the tool is part of the SQL Server 2008 tools, it is perfectly capable of diagnosing SQL Server 2005 deployments.</font>

<h2 style="margin: auto 0in">
  <font face="Times New Roman">Syntax</font>
</h2>

<pre><font size="1">SSBDIAGNOSE</font></pre>

<pre><font size="1"><span>        </span>[&lt;GeneralOptions&gt;] {&lt;Configuration&gt;}</font></pre>

<pre><font size="1"><span>        </span>|</font></pre>

<pre><font size="1"><span>        </span>[-?]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt;GeneralOptions&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>[-XML]</font></pre>

<pre><font size="1"><span>        </span>[-LEVEL {ERROR | WARNING | INFO}]</font></pre>

<pre><font size="1"><span>        </span>[&lt;ConnectionOptions&gt;]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt; Configuration&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>CONFIGURATION<span>  </span>{&lt;FromService&gt; &lt;ToService&gt; | &lt;FromService&gt; | &lt;ToService&gt;}</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1"><span>                       </span>{ ON CONTRACT contract_name }</font></pre>

<pre><font size="1"><span>                       </span>[ ENCRYPTION {ON | OFF | ANONYMOUS} ]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt;FromService&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>FROM SERVICE service_name [&lt;ConnectionOptions&gt;] [&lt;MirrorOptions&gt;]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt;ToService&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>TO SERVICE service_name [,broker_id]</font></pre>

<pre><font size="1"><span>        </span>[&lt;ConnectionOptions&gt;] [&lt;MirrorOptions&gt;]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt;MirrorOptions&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>MIRROR &lt;ConnectionOptions&gt;</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">&lt;ConnectionOptions&gt; ::=</font></pre>

<pre><font size="1"><span>        </span>{-E |<span>   </span>{-U login_id [-P password]}}</font></pre>

<pre><font size="1"><span>        </span>[-S server_name[\instance_name]]</font></pre>

<pre><font size="1"><span>        </span>[-d database_name]</font></pre>

<pre><font size="1"><span>        </span>[-l login_timeout]</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">SSBDIAGNOSE analyzes the configuration between two SQL Server Service Broker</font></pre>

<pre><font size="1">services, or for a single service. Any errors found are reported either in the</font></pre>

<pre><font size="1">command prompt window in human-readable text, or in formatted XML that can be</font></pre>

<pre><font size="1">redirected to a file or another application.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-XML<span>                        </span><span>    </span>Specifies generating the output as an XML file</font></pre>

<pre><font size="1"><span>                                </span>that can be redirected to a file or another</font></pre>

<pre><font size="1"><span>                                </span>application. If -XML is not specified, the</font></pre>

<pre><font size="1"><span>                                </span>output is displayed in the command prompt</font></pre>

<pre><font size="1"><span>                                </span>window.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-LEVEL<span>  </span>ERROR<span>                   </span>Display only error messages.</font></pre>

<pre><font size="1"><span>        </span>WARNING<span>                 </span>Display error and warning messages (default).</font></pre>

<pre><font size="1"><span>        </span>INFO<span>                    </span>Display error, warning, and informational</font></pre>

<pre><font size="1"><span>                                </span>messages.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">CONFIGURATION<span>                   </span>Requests a report of any configuration errors</font></pre>

<pre><font size="1"><span>                                </span>between two Service Broker services, or for a</font></pre>

<pre><font size="1"><span>                                </span>single service.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">ON CONTRACT contract_name<span>       </span>Requests that only configurations that use the</font></pre>

<pre><font size="1"><span>                                </span>specified contract be analyzed. If ON CONTRACT</font></pre>

<pre><font size="1"><span>                                </span>is not specified, ssbdiagnose reports on the</font></pre>

<pre><font size="1"><span>           </span><span>                     </span>contract named DEFAULT.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">ENCRYPTION ON<span>                   </span>Requests verification that full dialog security</font></pre>

<pre><font size="1"><span>                                </span>is configured (default).</font></pre>

<pre><font size="1"><span>           </span>OFF<span>                  </span>Requests verification that no dialog security is</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1"><span>                                </span>configured.</font></pre>

<pre><font size="1"><span>           </span>ANONYMOUS<span>            </span>Requests verification that anonymous dialog</font></pre>

<pre><font size="1"><span>                                </span>security is configured.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">FROM SERVICE service_name<span>       </span>Specifies the name of the service that initiates</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1"><span>                                </span>conversations.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">TO SERVICE service_name<span>         </span>Specifies the name of the target service.</font></pre>

<pre><font size="1"><span>           </span>[, broker_id]<span>        </span>Specifies the Service Broker identifier for the</font></pre>

<pre><font size="1"><span>                                </span>target database; broker_ID is a GUID.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">MIRROR<span>                          </span>Specifies that the associated Service Broker</font></pre>

<pre><font size="1"><span>                                </span>service is hosted in a mirrored database, and</font></pre>

<pre><font size="1"><span>                                </span>gives the connection information to the mirror</font></pre>

<pre><font size="1"><span>                                </span>database.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-E<span>                              </span>Opens a Windows Authentication connection to an</font></pre>

<pre><font size="1"><span>                                </span>instance of the SQL Server Database Engine using</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1"><span>                                </span>your Windows account. By default, if neither -E</font></pre>

<pre><font size="1"><span>                                </span>or -U are specified, ssbdiagnose uses Windows</font></pre>

<pre><font size="1"><span>                                </span>Authentication.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-U login_id<span>                     </span>Opens a SQL Server Authentication connection to</font></pre>

<pre><font size="1"><span>             </span><span>                   </span>an instance of the Database Engine using the</font></pre>

<pre><font size="1"><span>                                </span>specified login ID.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-P password<span>                     </span>Specifies the password for the login that is</font></pre>

<pre><font size="1"><span>                                </span>specified in -U.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-S server_name[\instance_name]<span>  </span>Specifies the instance of the Database Engine</font></pre>

<pre><font size="1"><span>                                </span>holding the Service Broker service to be</font></pre>

<pre><font size="1"><span>                                </span>analyzed. Specify server_name to connect to the</font></pre>

<pre><font size="1"><span>                                </span>default instance on the specified computer.</font></pre>

<pre><font size="1"><span>                                </span>Specify server_name\instance_name to connect to</font></pre>

<pre><font size="1"><span>                                </span>a named instance. If -S is not specified,</font></pre>

<pre><font size="1"><span>                                </span>ssbdiagnose connects to the default instance on</font></pre>

<pre><font size="1"><span>                                </span>the local computer.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-d database_name<span>                </span>Specifies the name of the database that is</font></pre>

<pre><font size="1"><span>                                </span>holding the service to be analyzed. If -d is</font></pre>

<pre><font size="1"><span>                                </span>not specified, the default is your login's</font></pre>

<pre><font size="1"><span>                                </span>default-database property.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-l login_timeout<span>                </span>Specifies the number of seconds before an</font></pre>

<pre><font size="1"><span>                                </span>attempt to connect times out. The value must be</font></pre>

<pre><font size="1"> <span>                               </span>between 1 and 65534. Not specifying -l or</font></pre>

<pre><font size="1"><span>                                </span>setting login_timeout 0 specifies the time-out</font></pre>

<pre><font size="1"><span>                                </span>to be infinite.</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">-?<span>                              </span>Displays command-line help.</font></pre>

<font face="Times New Roman">Now that’s one complex command line set of options! Actually, it is much easier to think in terms of the BEGIN DIALOG verb options. Lets compare the two side by side when establishing a dialog between a service named &#8216;initiator&#8217; hosted in the database db_init on ServerA and a service named &#8216;target&#8217; hosted in database db_target on ServerB:</font>

<table border="1" cellPadding="0" style="border: #bbbbbb 1pt dashed" class="MsoNormalTable">
  <tr>
    <td width="50%" style="width: 50%; background-color: transparent; border: #bbbbbb 1pt dashed; padding: 0.75pt">
      <p style="margin: 0in 0in 0pt" class="MsoNormal">
        <font size="3" face="Times New Roman">BEGIN DIALOG</font>
      </p>
    </td>
    
    <td style="background-color: transparent; border: #bbbbbb 1pt dashed; padding: 0.75pt">
      <p style="margin: 0in 0in 0pt" class="MsoNormal">
        <font size="3" face="Times New Roman">ssbdiagnose.exe</font>
      </p>
    </td>
  </tr>
  
  <tr>
    <td style="background-color: transparent; border: #bbbbbb 1pt dashed; padding: 0.75pt">
      <pre><font size="1">BEGIN DIALOG @handle</font></pre>
      
      <pre><font size="1"><span>    </span>FROM SERVICE [initiator]</font></pre>
      
      <pre><font size="1"><span>    </span>TO SERVICE 'target'</font></pre>
      
      <pre><font size="1"><span>    </span>ON CONTRACT [mycontract]</font></pre>
      
      <pre><font size="1"><span>    </span>WITH ENCRYPTION = OFF</font></pre>
    </td>
    
    <td style="background-color: transparent; border: #bbbbbb 1pt dashed; padding: 0.75pt">
      <pre><font size="1">ssbdiagnose configuration</font></pre>
      
      <pre><font size="1"><span>   </span>from service initiator -&lt;st1 :place w:st="on">S ServerA&lt;/st1> -d db_init</font></pre>
      
      <pre><font size="1"><span>   </span>to service target -S Server B -d db_target</font></pre>
      
      <pre><font size="1"><span>   </span>on contract mycontract</font></pre>
      
      <pre><font size="1"><span>   </span>encryption off</font></pre>
    </td>
  </tr>
</table>

<font face="Times New Roman">When the tool is run it will analyze the configuration of your services and endpoints and will report any problem it finds that would prevent the two services from being able to exchange messages. I have set up two services for test on my SQL Server instances and ran the tool and here is a sample diagnostic output:</font>

<pre><font size="1">C:\Program Files\Microsoft SQL Server\100\Tools\Binn</font></pre>

<pre><font size="1">&gt;SSBDiagnose.exe configuration from service "//MyInitiatorService" -S . -d diagn</font></pre>

<pre><font size="1">ose_init to service "//MyTargetService" -S .\katmai -d diagnose_target on contra</font></pre>

<pre><font size="1">ct mycontract encryption off</font></pre>

<pre><font size="1"> 29931 REMUSR10<span>        </span>diagnose_init<span>   </span>There is no route for service //MyTargetS</font></pre>

<pre><font size="1">ervice</font></pre>

<pre><font size="1"> 29931 REMUSR10\katmai diagnose_target There is no route for service //MyInitiat</font></pre>

<pre><font size="1">orService and broker instance 05D10DB7-8123-4F9F-AE7E-C69AF799410A</font></pre>

<pre><font size="1"> 29975 //MyTargetService diagnose_target User public does not have SEND permissi</font></pre>

<pre><font size="1">on on service //MyTargetService</font></pre>

<pre><font size="1"> 29950 REMUSR10\katmai<span>                 </span>The Service Broker endpoint was not found</font></pre>

<pre>&lt;o :p><font size="1"> </font>&lt;/o></pre>

<pre><font size="1">4 Errors, 0 Warnings</font></pre>

<font face="Times New Roman">In this case the tool has reported that the routes are missing or incorrect, the target service does not have the necessary SEND permission needed for dialogs started without encryption and finally the instance REMUSR10\Katmai does not have a Service Broker endpoint. The output contains an error code, the SQL Server instance to which the error applies, the database to which the error applies and the error message. The database name is optional and if missing it means the error message is relevant to the instance, not to a specific database. Optionally the output can be formatted as XML so that it can be processed by other automation tools. This is achieved by specifying the /XML command line option.</font>