---
id: 335
title: Using XSLT to generate Performance Counters code
date: 2009-04-10T22:50:47+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/04/10/308-revision-22/
permalink: /2009/04/10/308-revision-22/
---
Whenever I&#8217;m faced with a project in which I have to create a lot of tedious and repeating code I turn to the power of XML and XSLT. Rather than copy/paste the same code over and over again, just to end up with a refactoring and maintenance nightmare, I create an XML definition file and an XSLT transformation. I am then free to add new elements to the XML definition or to change the way the final code is generated from the XSLT transformation. This can be fully integrated with Visual Studio so that the code generation happens at project build time and the environment shows the generated code as a dependency of the XML definition file.

A few examples of how I&#8217;m using this code generation via XSLT are:

Data Access Layer
:   I know this will raise quite a few eyebrows, but I always write my own data access layer from scratch and is generated via XSLT.

Performance Counters
:   I create all my performance counters objects via XSLT generation, automating the process of defining/installing them and the access to emit and consume the counter values.

Wire Frames
:   In any project that has networking communication I access the wire format from classes generated via XSLT that take care of serialization and validation.

For example I&#8217;ll show how to create a class library that can be added to your project to expose Performance Counters from your application.

### MSXSL.EXE

To start you&#8217;ll need to download the <a href="http://www.microsoft.com/downloads/details.aspx?FamilyId=2FB55371-C94E-4373-B0E9-DB4816552E41&#038;displaylang=en" target="_blank">Command Line Transformation Utility (msxsl.exe)</a>. This is a command line tool that takes an XML and an XSLT file as input and produces the result transformation. As a side not I actually use the same technique on non-Windows platforms, but there of course I use the <a href="http://www.xmlsoft.org/" target="_blank">xsltproc</a> utility.

msxsl.exe has to be placed in a convenient location from where it is accessible by Visual Studio. Download the file and place it in Visual Studio&#8217;s CommonIDE folder because the build environment has a macro for it: <tt>$DevEnvDir</tt>.

### The Project

Create a new project in Visual Studio. Choose a C# Class Library type project and save it as &#8220;PerformanceCounters&#8221;. After the project is created, add to it two new files, from menu <tt>Project/Add New Item</tt> or press <tt>Ctrl+Shift+A</tt>. For the first file choose an <tt>XML file</tt> type and name it <tt>Counters.xml</tt>. For the second one choose an <tt>XSLT file</tt> type and name it <tt>Counters.xslt</aa>.</p> 

<p>
  Next add the step that actually performs XSLT transformation to the project build process. Go to the project properties, either from menu <tt>Project/PerformcanceCounters properties...</tt> or right-click the project in the Solutions Explorer and select <tt>Properties</tt> from the context menu. Choose the <tt>Build Events</tt> pane in the right side navigation tab to get to the pre-build and post-build project events. Click the <tt>Edit pre-build...</tt> button and add the following command:
</p>

<pre>"$(DevEnvDir)msxsl.exe" "$(ProjectDir)Counters.xml" "$(ProjectDir)Counters.xslt"
-o "$(ProjectDir)CountersGenerated.cs"
</pre>

<p>
  This command will run the command line transformation tool (msxsl.exe) and generate the <tt>CountersGenerated.cs</tt> file each time the project is built, prior to the project being actually built.
</p>

<p>
  <a href="http://test.rusanu.com/wp-content/uploads/2009/04/prebuildevent.png"><img src="http://test.rusanu.com/wp-content/uploads/2009/04/prebuildevent.png" alt="" title="prebuildevent" width="150" class="alignnone size-thumbnail wp-image-314" /></a>
</p>

<p>
  Save and close the project properties. The last thing we need now is to add the generated <tt>CountersGenerated.cs</tt> file to our project. However, we do not want this to be treated as an ordinary source file, we want the IDE to recognize that is a generated file dependent of the <tt>Counters.xml</tt> file. The way I prefer to do this is to actually manually add it directly into the project definition file. Open the PerformanceCounters.csproj file in your editor, locate the <tt><ItemGroup></tt> containing the <tt>Program.cs</tt> and insert this snippet:
</p>

<pre>
	  &lt;Compile Include="CountersGenerated.cs"&gt;
		  &lt;AutoGen&gt;True&lt;/AutoGen&gt;
		  &lt;DependentUpon&gt;Counters.xml&lt;/DependentUpon&gt;
	  &lt;/Compile&gt;
</pre>

<p>
  <a href="http://test.rusanu.com/wp-content/uploads/2009/04/editproject.png"><img src="http://test.rusanu.com/wp-content/uploads/2009/04/editproject.png" alt="" title="editproject" width="300" height="90" class="alignnone size-medium wp-image-327" /></a>
</p>

<p>
  Save the manually edited .csproj file and open the project normally from Visual Studio. The <tt>CountersGenerated.cs</tt> file now shows up in the Project Explorer as a generated file depending on <tt>Counters.xml</tt>:
</p>

<p>
  <a href="http://test.rusanu.com/wp-content/uploads/2009/04/dependentautogen.png"><img src="http://test.rusanu.com/wp-content/uploads/2009/04/dependentautogen.png" alt="" title="dependentautogen" width="255" height="173" class="alignnone size-medium wp-image-329" /></a>
</p>

<p>
  The warning triangle is shown because the file does not exist on the disk, but don't worry about it as it will be generated shortly.
</p>

<h3>
  Defining the Performance Counters
</h3>

<p>
  We can go ahead and define the Performance Counters. The counters covering related functionality are grouped together and the groups can be single instance or multiple instance types. I also like to create a manager class that controls the management of the counters taking care of things like deployment during application setup and loading the counters when application starts. So my choice is for a XML like <tt>manager/group/counter</tt>. Some attributes are needed for the code generation XSLT transformation to know what class names and namespaces to use and some attributes are Performance Counters specific, like the counter types and bases for average counters. For start we'll create a simple counter and a group:
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">&lt;?</span><span style="color:Maroon">xml</span><span style="color:Blue">&nbsp;</span><span style="color:Red">version</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">1.0</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">encoding</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">utf-8</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;?&gt;<br />
&lt;</span><span style="color:Maroon">manager</span><span style="color:Blue">&nbsp;</span><span style="color:Red">xmlns</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">http://rusanu.com/Schemas/PerformanceCounters/v1.0</span><span style="color:Black">"<br />
</span><span style="color:Blue">		</span><span style="color:Red">clrname</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">PerformanceCountersManager</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">namespace</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">PerformanceCounters</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
	&lt;</span><span style="color:Maroon">group</span><span style="color:Blue">&nbsp;</span><span style="color:Red">name</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">My&nbsp;Counters</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">clrname</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">MyCounters</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">type</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">SingleInstance</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
		&lt;</span><span style="color:Maroon">description</span><span style="color:Blue">&gt;</span><span style="color:Black">Demo&nbsp;Counters&nbsp;for&nbsp;XSLT&nbsp;code&nbsp;generation&nbsp;project</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">description</span><span style="color:Blue">&gt;<br />
		&lt;</span><span style="color:Maroon">counter</span><span style="color:Blue">&nbsp;</span><span style="color:Red">name</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">My&nbsp;Count</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">type</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">NumberOfItems32</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span>
                                      <span style="color:Red">clrname</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">Count</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">hasbase</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">No</span><span style="color:Black">"</span><span style="color:Blue">&gt;</span><span style="color:Black">Demo&nbsp;Counter</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">counter</span><span style="color:Blue">&gt;<br />
	&lt;/</span><span style="color:Maroon">group</span><span style="color:Blue">&gt;<br />
&lt;/</span><span style="color:Maroon">manager</span><span style="color:Blue">&gt;</span>
</pre>

<p>
  This definition will create:
</p>

<ul>
  <li>
    A manager class named <tt>PerformanceCounters.PerformanceCountersManager</tt>.
  </li>
  <li>
    A performance counters category named <tt>My Group</tt>.
  </li>
  <li>
    A C# class named <tt>MyGroup</tt> for the above performance counters category.
  </li>
  <li>
    A performance counter named <tt>My Count</tt>.
  </li>
  <li>
    A C# method named <tt>IncrementMyCount</tt> to increment the counter above.
  </li>
</ul>