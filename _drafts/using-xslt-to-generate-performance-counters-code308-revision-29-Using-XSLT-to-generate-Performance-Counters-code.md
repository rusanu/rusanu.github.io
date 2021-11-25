---
id: 342
title: Using XSLT to generate Performance Counters code
date: 2009-04-11T00:12:47+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/04/11/308-revision-29/
permalink: /2009/04/11/308-revision-29/
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

To start you&#8217;ll need to download the <a href="http://www.microsoft.com/downloads/details.aspx?FamilyId=2FB55371-C94E-4373-B0E9-DB4816552E41&#038;displaylang=en" target="_blank">Command Line Transformation Utility (msxsl.exe)</a>. This is a command line tool that takes an XML and an XSLT file as input and produces the result transformation. As a side note I actually use the same technique on non-Windows platforms, but there of course I use the <a href="http://www.xmlsoft.org/" target="_blank">xsltproc</a> utility.

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
                                      <span style="color:Red">clrname</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">MyCount</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">hasbase</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">No</span><span style="color:Black">"</span><span style="color:Blue">&gt;</span><span style="color:Black">Demo&nbsp;Counter</span><span style="color:Blue">&lt;/</span><span style="color:Maroon">counter</span><span style="color:Blue">&gt;<br />
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

<p>
  Now we need to set in place the actual XSLT transformation that will generate code as we desire.
</p>

<h3>
  The XSTL transformation
</h3>

<p>
  We want our XSLT transformation to generate C# code, so we're going to start by a stylesheet that will generate the skeleton of a C# file: the using directives and some comment to warn the user that the file is a generated file and should not be manually modified. Also our XSLT transformation is directed to generate an <tt>text</tt> output as opposed to the default <tt>XML</tt> output:
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">&lt;?</span><span style="color:Maroon">xml</span><span style="color:Blue">&nbsp;</span><span style="color:Red">version</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">1.0</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">encoding</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">UTF-8</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;?&gt;<br />
&lt;</span><span style="color:Teal">xsl:stylesheet</span><span style="color:Blue">&nbsp;</span><span style="color:Red">version</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">1.0</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;<br />
	</span><span style="color:Red">xmlns:xsl</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">http://www.w3.org/1999/XSL/Transform</span><span style="color:Black">"<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Red">xmlns:pc</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">http://rusanu.com/Schemas/PerformanceCounters/v1.0</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
	&lt;</span><span style="color:Teal">xsl:output</span><span style="color:Blue">&nbsp;</span><span style="color:Red">method</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">text</span><span style="color:Black">"</span><span style="color:Blue">&nbsp;</span><span style="color:Red">encoding</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">utf-8</span><span style="color:Black">"</span><span style="color:Blue">/&gt;<br />
	&lt;</span><span style="color:Teal">xsl:template</span><span style="color:Blue">&nbsp;</span><span style="color:Red">match</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">/</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">/*&nbsp;This&nbsp;file&nbsp;was&nbsp;automatically&nbsp;generated&nbsp;during&nbsp;project&nbsp;build.<br />
Do&nbsp;not&nbsp;manually&nbsp;modify&nbsp;this&nbsp;file&nbsp;as&nbsp;updates&nbsp;will&nbsp;be&nbsp;lost.*/<br />
using&nbsp;System;<br />
using&nbsp;System.Diagnostics;<br />
<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:apply-templates</span><span style="color:Blue">/&gt;<br />
&lt;/</span><span style="color:Teal">xsl:template</span><span style="color:Blue">&gt;<br />
&lt;/</span><span style="color:Teal">xsl:stylesheet</span><span style="color:Blue">&gt;</span>
</pre>

<p>
  This skeleton XSLT can be used for any C# code generation, provided of course you modify the appropriate <tt>using</tt> directives.
</p>

<p>
  Next we should add a template to generate our C# code. How one writes XSLT transformation templates is largely a matter of style and once you get the hang of it it quickly becomes a second nature. XSLT is a fully fledged programming language of its own and you can start thinking at templates in terms of procedures and functions. At least I know I do :). Our XSLT template should do the following:
</p>

<ul>
  <li>
    Identify the manager defined in the XML file and generate a C# class for it.
  </li>
  <li>
    Generate a class for each performance counter category.
  </li>
  <li>
    Generate a method for each individual performance counter value to read and increment the counter.
  </li>
  <li>
    Hook up the manager class to provide methods for deploying and deleting the performance counter categories.
  </li>
  <li>
    Hook up the manager class with an instance of each performance counter category class.
  </li>
  <li>
    Hook up the manager to provide loading of the default instance of the performance counters category classes.
  </li>
</ul>

<p>
  Note that the above refer mostly to the SingleInstance performance counter category types. Multiple Instance category are a bitmore complex and I will cover them in my next post.
</p>

<pre>
<span style="color: Black"></span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:template</span><span style="color:Blue">&nbsp;</span><span style="color:Red">match</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">/pc:manager</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">namespace&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@namespace</span><span style="color:Black">"</span><span style="color:Blue">/&gt;<br />
</span><span style="color:Black">{<br />
public&nbsp;partial&nbsp;class&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;<br />
</span><span style="color:Black">{<br />
</span><span style="color:Blue">	&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:group[@type='SingleInstance']</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	private&nbsp;static&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;__</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;=&nbsp;<br />
		new&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">();<br />
	public&nbsp;static&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;Default</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;{<br />
		get&nbsp;{return&nbsp;__</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">;&nbsp;}&nbsp;<br />
	}<br />
</span><span style="color:Blue">	&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
	<br />
</span><span style="color:Black">	public&nbsp;static&nbsp;void&nbsp;LoadSingleInstance(bool&nbsp;fReadOnly)<br />
	{<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:group[@type='SingleInstance']</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">		__</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">.Load(fReadOnly);<br />
</span><span style="color:Blue">		&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	}		<br />
</span><span style="color:Blue">	<br />
</span><span style="color:Black">	public&nbsp;static&nbsp;void&nbsp;DeleteSingleInstance()<br />
	{<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:group[@type='SingleInstance']</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
		&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">.Delete();<br />
</span><span style="color:Blue">		&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	}<br />
</span><span style="color:Blue">	<br />
</span><span style="color:Black">	public&nbsp;static&nbsp;void&nbsp;CreateSingleInstance()<br />
	{<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:group[@type='SingleInstance']</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
		&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">.Create();<br />
</span><span style="color:Blue">		&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	}	<br />
}<br />
<br />
</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:group</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">public&nbsp;partial&nbsp;class&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;<br />
</span><span style="color:Black">{<br />
	private&nbsp;const&nbsp;string&nbsp;__description&nbsp;=&nbsp;<br />
		@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:description/text()</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">";<br />
	private&nbsp;const&nbsp;string&nbsp;__category&nbsp;=&nbsp;<br />
		@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">";<br />
</span><span style="color:Blue">	<br />
	&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:counter</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;	private&nbsp;PerformanceCounter&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">;<br />
</span><span style="color:Blue">	&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@hasbase='Yes'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	private&nbsp;PerformanceCounter&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base;<br />
</span><span style="color:Blue">	&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
<br />
</span><span style="color:Black">	public&nbsp;void&nbsp;Increment</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">(long&nbsp;value)<br />
	{<br />
		_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">.IncrementBy(value);<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@hasbase='Yes'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">		_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base.IncrementBy(1);<br />
</span><span style="color:Blue">		&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">	}<br />
</span><span style="color:Blue">	<br />
</span><span style="color:Black">	public&nbsp;float&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;{<br />
		get&nbsp;{&nbsp;return&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">.NextValue();&nbsp;}&nbsp;<br />
	}<br />
</span><span style="color:Blue">	<br />
	&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
	<br />
	&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@type='MultipleInstance'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;internal&nbsp;void&nbsp;Load(bool&nbsp;fReadOnly,&nbsp;string&nbsp;instanceName)&nbsp;{<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:counter</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;=&nbsp;new&nbsp;PerformanceCounter(<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__category,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">",<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;instanceName,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fReadOnly);<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@hasbase='Yes'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base&nbsp;=&nbsp;new&nbsp;PerformanceCounter(<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__category,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base",<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;instanceName,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fReadOnly);<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;}<br />
</span><span style="color:Blue">	&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
<br />
	&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@type='SingleInstance'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;internal&nbsp;void&nbsp;Load&nbsp;(bool&nbsp;fReadOnly)<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:counter</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">&nbsp;=&nbsp;new&nbsp;PerformanceCounter(<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__category,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">",<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fReadOnly);<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@hasbase='Yes'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@clrname</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base&nbsp;=&nbsp;new&nbsp;PerformanceCounter(<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__category,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base",<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fReadOnly);<br />
</span><span style="color:Blue">&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;&nbsp;&nbsp;&nbsp;&nbsp;&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;&nbsp;}	<br />
</span><span style="color:Blue">	&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
	<br />
</span><span style="color:Black">&nbsp;&nbsp;&nbsp;internal&nbsp;static&nbsp;bool&nbsp;Exists<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		get&nbsp;{return&nbsp;PerformanceCounterCategory.Exists(__category);}<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;internal&nbsp;static&nbsp;void&nbsp;Delete&nbsp;()<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		PerformanceCounterCategory.Delete(__category);<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;public&nbsp;static&nbsp;void&nbsp;Create&nbsp;()<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		CounterCreationDataCollection&nbsp;ccdc&nbsp;=&nbsp;new&nbsp;CounterCreationDataCollection();<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">./pc:counter</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">		ccdc.Add(new&nbsp;CounterCreationData&nbsp;(<br />
			@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">",<br />
			@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">.</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">",<br />
			PerformanceCounterType.</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@type</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">));<br />
</span><span style="color:Blue">		&lt;</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&nbsp;</span><span style="color:Red">test</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@hasbase='Yes'</span><span style="color:Black">"</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">		ccdc.Add(new&nbsp;CounterCreationData&nbsp;(<br />
			@"</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@name</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">Base",<br />
			@"Base&nbsp;for&nbsp;</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">.</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">",<br />
			PerformanceCounterType.AverageBase));<br />
</span><span style="color:Blue">		&lt;/</span><span style="color:Teal">xsl:if</span><span style="color:Blue">&gt;<br />
		&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">		PerformanceCounterCategory.Create(<br />
			__category,<br />
			__description,<br />
			PerformanceCounterCategoryType.</span><span style="color:Blue">&lt;</span><span style="color:Teal">xsl:value-of</span><span style="color:Blue">&nbsp;</span><span style="color:Red">select</span><span style="color:Blue">=</span><span style="color:Black">"</span><span style="color:Blue">@type</span><span style="color:Black">"</span><span style="color:Blue">/&gt;</span><span style="color:Black">,<br />
			ccdc);<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
}<br />
</span><span style="color:Blue">&nbsp;<br />
&lt;/</span><span style="color:Teal">xsl:for-each</span><span style="color:Blue">&gt;<br />
</span><span style="color:Black">}<br />
</span><span style="color:Blue">&lt;/</span><span style="color:Teal">xsl:template</span><span style="color:Blue">&gt;</span>
</pre>

<h3>
  The generate code
</h3>

<p>
  If we go ahead and build the project. It will run our XSLT transformation over the definition XML file and the result will be saved in the <tt>CountersGenerated.cs</tt> file:
</p>

<pre>
<span style="color: Black"></span><span style="color:Green">/*&nbsp;This&nbsp;file&nbsp;was&nbsp;automatically&nbsp;generated&nbsp;during&nbsp;project&nbsp;build.<br />
Do&nbsp;not&nbsp;manually&nbsp;modify&nbsp;this&nbsp;file&nbsp;as&nbsp;updates&nbsp;will&nbsp;be&nbsp;lost.*/<br />
</span><span style="color:Blue">using</span><span style="color:Black">&nbsp;System;<br />
</span><span style="color:Blue">using</span><span style="color:Black">&nbsp;System.Diagnostics;<br />
<br />
<br />
</span><span style="color:Blue">namespace</span><span style="color:Black">&nbsp;PerformanceCounters<br />
{<br />
</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">partial</span><span style="color:Black">&nbsp;</span><span style="color:Blue">class</span><span style="color:Black">&nbsp;</span><span style="color:Teal">PerformanceCountersManager<br />
</span><span style="color:Black">{<br />
	<br />
	</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Teal">MyCounters</span><span style="color:Black">&nbsp;__MyCounters&nbsp;=&nbsp;<br />
		</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:Teal">MyCounters</span><span style="color:Black">();<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Teal">MyCounters</span><span style="color:Black">&nbsp;DefaultMyCounters&nbsp;{<br />
		</span><span style="color:Blue">get</span><span style="color:Black">&nbsp;{</span><span style="color:Blue">return</span><span style="color:Black">&nbsp;__MyCounters;&nbsp;}&nbsp;<br />
	}<br />
	<br />
	<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;LoadSingleInstance(</span><span style="color:Blue">bool</span><span style="color:Black">&nbsp;fReadOnly)<br />
	{<br />
		<br />
		__MyCounters.Load(fReadOnly);<br />
		<br />
	}		<br />
	<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;DeleteSingleInstance()<br />
	{<br />
		</span><span style="color:Teal">MyCounters</span><span style="color:Black">.Delete();<br />
		<br />
	}<br />
	<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;CreateSingleInstance()<br />
	{<br />
		</span><span style="color:Teal">MyCounters</span><span style="color:Black">.Create();<br />
		<br />
	}	<br />
}<br />
<br />
<br />
</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">partial</span><span style="color:Black">&nbsp;</span><span style="color:Blue">class</span><span style="color:Black">&nbsp;</span><span style="color:Teal">MyCounters<br />
</span><span style="color:Black">{<br />
	</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Blue">const</span><span style="color:Black">&nbsp;</span><span style="color:Blue">string</span><span style="color:Black">&nbsp;__description&nbsp;=&nbsp;<br />
		</span><span style="color:Maroon">@"Demo&nbsp;Counters&nbsp;for&nbsp;XSLT&nbsp;code&nbsp;generation&nbsp;project"</span><span style="color:Black">;<br />
	</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Blue">const</span><span style="color:Black">&nbsp;</span><span style="color:Blue">string</span><span style="color:Black">&nbsp;__category&nbsp;=&nbsp;<br />
		</span><span style="color:Maroon">@"My&nbsp;Counters"</span><span style="color:Black">;<br />
	<br />
	<br />
&nbsp;	</span><span style="color:Blue">private</span><span style="color:Black">&nbsp;</span><span style="color:Teal">PerformanceCounter</span><span style="color:Black">&nbsp;_MyCount;<br />
	<br />
<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;IncrementMyCount(</span><span style="color:Blue">long</span><span style="color:Black">&nbsp;value)<br />
	{<br />
		_MyCount.IncrementBy(value);<br />
		<br />
	}<br />
	<br />
	</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">float</span><span style="color:Black">&nbsp;MyCount&nbsp;{<br />
		</span><span style="color:Blue">get</span><span style="color:Black">&nbsp;{&nbsp;</span><span style="color:Blue">return</span><span style="color:Black">&nbsp;_MyCount.NextValue();&nbsp;}&nbsp;<br />
	}<br />
	<br />
	<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">internal</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Load&nbsp;(</span><span style="color:Blue">bool</span><span style="color:Black">&nbsp;fReadOnly)<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
&nbsp;&nbsp;&nbsp;&nbsp;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;_MyCount&nbsp;=&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:Teal">PerformanceCounter</span><span style="color:Black">(<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;__category,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Maroon">@"My&nbsp;Count"</span><span style="color:Black">,<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;fReadOnly);<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<br />
&nbsp;&nbsp;&nbsp;&nbsp;}	<br />
	<br />
	<br />
&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">internal</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">bool</span><span style="color:Black">&nbsp;Exists<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		</span><span style="color:Blue">get</span><span style="color:Black">&nbsp;{</span><span style="color:Blue">return</span><span style="color:Black">&nbsp;</span><span style="color:Teal">PerformanceCounterCategory</span><span style="color:Black">.Exists(__category);}<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">internal</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Delete&nbsp;()<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		</span><span style="color:Teal">PerformanceCounterCategory</span><span style="color:Black">.Delete(__category);<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
<br />
&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">public</span><span style="color:Black">&nbsp;</span><span style="color:Blue">static</span><span style="color:Black">&nbsp;</span><span style="color:Blue">void</span><span style="color:Black">&nbsp;Create&nbsp;()<br />
&nbsp;&nbsp;&nbsp;&nbsp;{<br />
		</span><span style="color:Teal">CounterCreationDataCollection</span><span style="color:Black">&nbsp;ccdc&nbsp;=&nbsp;<br />
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:Teal">CounterCreationDataCollection</span><span style="color:Black">();<br />
		<br />
		ccdc.Add(</span><span style="color:Blue">new</span><span style="color:Black">&nbsp;</span><span style="color:Teal">CounterCreationData</span><span style="color:Black">&nbsp;(<br />
			</span><span style="color:Maroon">@"My&nbsp;Count"</span><span style="color:Black">,<br />
			</span><span style="color:Maroon">@"Demo&nbsp;Counter"</span><span style="color:Black">,<br />
			</span><span style="color:Teal">PerformanceCounterType</span><span style="color:Black">.NumberOfItems32));<br />
		<br />
		</span><span style="color:Teal">PerformanceCounterCategory</span><span style="color:Black">.Create(<br />
			__category,<br />
			__description,<br />
			</span><span style="color:Teal">PerformanceCounterCategoryType</span><span style="color:Black">.SingleInstance,<br />
			ccdc);<br />
&nbsp;&nbsp;&nbsp;&nbsp;}<br />
}<br />
&nbsp;<br />
<br />
}<br />
</span>
</pre>

<p>
  So what did we achieve? We could had written this code manually in about 5 minutes. But the big advantage comes as we add more counters to our XML definition file. We could expand it to have 10 categories with 15 counters each and it would still generate all the needed counters. And refactoring te code is a breeze. Don't like how the counters are created? Just change the XSLT stylesheet and rebuild the project, all the performance counters creation code wil match your new preference.
</p>

<p>
  In a future post I will cover how to use the performance counters in your projects, how to work with multiple instance counters and how to deal with the pesky <tt>AvgTimer32</tt> counter type.
</p>