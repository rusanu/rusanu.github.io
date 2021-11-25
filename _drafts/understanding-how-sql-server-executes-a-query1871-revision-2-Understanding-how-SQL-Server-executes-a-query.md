---
id: 1873
title: Understanding how SQL Server executes a query
date: 2013-06-14T03:02:21+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/06/14/1871-revision-2/
permalink: /2013/06/14/1871-revision-2/
---
This article is an introduction into how SQL Server works. My target audience are the developers and I hope this article will help you write better database code and will help you get starting when having to investigate performance problems in the database back end.

## Requests

SQL Server is a client-server platform. The only way to interact with the back-end database is by sending requests that contains commands for the database. The protocol used to communicate between your application and the database is called TDS (Tabular Data Sream) and is described on MSDN in the Technical Document <a href="http://msdn.microsoft.com/en-us/library/dd304523.aspx" target="_blank">[MS-TDS]: Tabular Data Stream Protocol</a>. thr application can use one of the several client-side implementations of the protocol: the CLR managed SqlClient, OleDB, ODBC, JDBC, PHP Driver for SQL Server or the open source FreeTDS implementation. The gist of it is that when your application whats the database to do _anything_ it will send a request over the TDS protocol. The request itself can take several forms:
:   <a href="http://msdn.microsoft.com/en-us/library/dd357447.aspx" target="_blank">Batch Request</a> 
    
    :   This request type contains just T-SQL text for a batch to be executed. This type of requests do not have parameters, but obviously the T-SQL batch itself wi can contain local variables declarations. This is the type of request SqlClient sends if you invoke any of the <a href="http://msdn.microsoft.com/en-us/library/hhxxcw8c.aspx" target="_blank"><tt>SqlCommand.ExecuteReader()<tt></a>/.<a href="" target="_blank">ExecuteNonQuery()/.ExecuteScalar()/.ExecuteXmlReader() (or they respective asyncronous equivalents) </dt> 
        
        <dd>
          <a href="http://msdn.microsoft.com/en-us/library/dd303353.aspx" target="_blank">Remote Procedure Call Request</a> 
          
          <dd>
            <dt>
            </dt>
            
            <dd>
              <a href="http://msdn.microsoft.com/en-us/library/dd357606.aspx" target="_blank">Bulk Load Request</a> 
              
              <dd>
                <dt>
                </dt>
                
                <p>
                  <!-- more -->
                </p>