---
id: 309
title: Using XSLT to generate Performance Counters code
date: 2009-04-09T18:36:18+00:00
author: remus
layout: revision
guid: http://rusanu.com/2009/04/09/308-revision/
permalink: /2009/04/09/308-revision/
---
Whenever I&#8217;m faced with a project in which I have to create a lot of tedious and repeating code I turn to the power of XML and XSLT. Rather than copy/paste the same code over and over again, just to end up with a refactoring and maintenance nightmare, I create an XML definition file and an XSLT transformation. I am then free to add new elements to the XML definition or to change the way the final code is generated from the XSLT transformation. This can be fully integrated with Visual Studio so that the code generation happens at project build time and the environment shows the generated code as a dependency of the XML definition file.

A few examples of how I&#8217;m using this code generation via XSLT are:

Data Access Layer
:   I know this will raise 

Performance Counters
:   I create all my performance counters objects via XSLT generation, automating the process of defining/installing them and the access to emit and consume the counter values.

: