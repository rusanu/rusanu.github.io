---
id: 310
title: Using XSLT to generate Performance Counters code
date: 2009-04-09T18:46:00+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/04/09/308-revision-2/
permalink: /2009/04/09/308-revision-2/
---
Whenever I&#8217;m faced with a project in which I have to create a lot of tedious and repeating code I turn to the power of XML and XSLT. Rather than copy/paste the same code over and over again, just to end up with a refactoring and maintenance nightmare, I create an XML definition file and an XSLT transformation. I am then free to add new elements to the XML definition or to change the way the final code is generated from the XSLT transformation. This can be fully integrated with Visual Studio so that the code generation happens at project build time and the environment shows the generated code as a dependency of the XML definition file.

A few examples of how I&#8217;m using this code generation via XSLT are:

Data Access Layer
:   I know this will raise quite a few eyebrows, but I always write my own data access layer from scratch and is generated via XSLT.

Performance Counters
:   I create all my performance counters objects via XSLT generation, automating the process of defining/installing them and the access to emit and consume the counter values.

Wire Frames
:   In any project that has networking communication I access the wire format from classes generated via XSLT that take care of serialization and validation.

### MSXSL.EXE

To start you&#8217;ll need to download the <a href="http://www.microsoft.com/downloads/details.aspx?FamilyId=2FB55371-C94E-4373-B0E9-DB4816552E41&#038;displaylang=en" target="_blank">Command Line Transformation Utility (msxsl.exe)</a>. This is a command line tool that takes an XML and an XSLT file as input and produces the result transformation. As a side not I actually use the same technique on non-Windows platforms, but there of course I use the <a href="http://www.xmlsoft.org/" target="_blank">xsltproc</a> utility.