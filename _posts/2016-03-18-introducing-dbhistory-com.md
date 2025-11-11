---
id: 2466
title: Introducing DBHistory.com
date: 2016-03-18T07:25:50+00:00
author: remus
layout: post
guid: /?p=2466
permalink: /2016/03/18/introducing-dbhistory-com/
categories:
  - Announcements
  - DBHistory.com
---
After 145 months of employment with Microsoft, at the beginning of March 2016 I quit the SQL Server team to pursue my dream of creating a better solution for SQL Server performance monitoring and configuration management. I am now dedicating all my time to building [DBHistory.com](http://dbhistory.com).

## What is DBHistory.com?

At first DBHistory.com will allow you to track the history of all configuration and schema changes on SQL Server. You will see when the change occurred, which user initiated it, on which server, in which database, what objects where affected, and the exact T-SQL text of the statement that triggered the change. Using DBHistory.com you will be able to quickly asses the history of an object, what changes occurred in a certain time interval, view all changes done by a particular user and so on.

There is no tool to purchase, no code to deploy, no storage to reserve, no contracts to sign. Only a configuration wizard to run, DBHistory.com uses SQL Server&#8217;s own Event Notifications mechanisms to push all change notifications from the monitored servers to the DBHistory.com service. The change notifications leave the monitored server immediately and are then safely stored with DBHistory.com.

You can use DBHistory.com if you are a software vendor that deploys a solution on-premise at your clients and need to know if any configuration or schema change occurred in your applications. You can use DBHistory.com if you are a remote DBA company that needs to know the history of changes on any of your client&#8217;s SQL Server databases, no matter who or when did the change. You can use DBHistory.com if you are a consultant that needs to know what changed since your last visit. You can use DBHistory.com if you have many employees with database access and want to know what changes were done by whom and when. You can use DBHistory.com if you are database development company that needs a history of changes done directly to the database in the development environment. You can use DBHistory.com if you distribute an application with free SQL Server Express Edition and want to understand how many times your application was deployed.

You will be able to set alarms to get notifications when certain changes occur, or when important objects are modified.

The road ahead will add more functionality to DBHistory.com, including performance monitoring

## When can I try DBHistory.com?

If you are interested, please sign up now at [DBHistory.com](http://dbhistory.com). Soon I will extend beta invites for those interested in trying out the service.