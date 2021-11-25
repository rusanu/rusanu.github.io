---
id: 944
title: 'TRY-CATCH-THROW: Exception handling in T-SQL'
date: 2010-11-13T23:24:43+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/11/13/940-revision-4/
permalink: /2010/11/13/940-revision-4/
---
When SQL Server 2005 introduced <TT>BEGIN TRY</TT> and <TT>BEGIN CATCH</TT> syntax, it was a huge improvement over the previous error handling based on @@ERROR check _after each statement_. Finally, T-SQL joined the rank of _programming_ languages, no more just a data access language. Experience has shown that exception handling leads to better code compared to error checks. Yes, SEH is slower, but is basically impossible to maintain the code discipline to check @@ERROR after every operation, so exception handling is just so much easier to get right. And besides, @@ERROR never had such a masterpiece article to guide you trough like <a href="http://www.microsoft.com/msj/0197/exception/exception.aspx" target="_blank">A Crash Course on the Depths of Win32™ Structured Exception Handling</a>.

But when trying to use the new TRY/CATCH exception handling in T-SQL code, one problem quickly became apparent: the CATCH block was masking the original error metadata: error number/severity/state, error text, origin line and so on. Within a CATCH block the code was only allowed to raise a \*new\* error. Sure, the original error information could be passed on in the raised error message, but only as a _message_. The all important error code was changed.

This may seem like a minor issue, but turns out to have a quite serious cascading effect: the caller now has to understand the _new_ error codes raised by your code, instead of the original system error codes.