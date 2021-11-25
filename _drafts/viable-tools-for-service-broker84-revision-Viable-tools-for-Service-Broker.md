---
id: 1171
title: Viable tools for Service Broker
date: 2008-05-16T10:05:39+00:00
author: remus
layout: revision
guid: http://rusanu.com/2008/05/16/84-revision/
permalink: /2008/05/16/84-revision/
---
Technically Service Broker can offer a huge value over any other comparable technology, but every project faces in the early stages the same barrier: the extremely steep learning curve faced by developers and dba in order deploy a Service Broker solution. As the adopters try to set up a simple scenario to &#8216;play&#8217; with the technology before making a decision they are facing an apparently insurmountable challenge: set up the configuration for that very first message to get through. Not only services, queue, contracts and message types have to be correctly configured, but also the transport security (endpoints and certificates, logins and endpoint connect permissions), service routing and last but not least service security (remote service bindings, again certificates, users without login and service send permission). These are all new concepts, not trivial ones, and they **_all_** have to be correct. And I haven&#8217;t even mentioned database master keys! So one is faced with the task of configuring more than ten new object types, never encountered before, just to test his first message. No wonder some feel frustrated and may abandon the technology before even seeing what is capable of. 

Tools are the biggest missing piece of this puzzle. A wizard that would walk the new developer or dba through the maze of new concepts and objects to help him get that very first test scenario to work would be tremendous help. Unfortunately the set of tools that ship with SQL Server is in dire need of such wonder. For years my recommendation was to give a try to the <a href="http://www.codeplex.com/slm" target="_blank">Service Listing Manager</a> tool. This tool, specially the command line version, takes all the headake out and can do all the work for you, configuring everything needed for two services to be able to exchange messages. But this tool is not an officially supported Microsoft release, and therefore many organizations cannot afford to use it in their environment.

Recently I have learned that 3rd party vendors stepped up to the challenge and have come up with a set of tools designed for Service Broker. Quest Software&#8217;s <a href="http://www.quest.com/toad-for-sql-server/" target="_blank">Toad</a> is a well known toolset, and on Oracle environment is **_the_** toolset, the one everybody uses. And now it contains the Service Broker Manager, a set of tools that covers everything related to Service Broker in SQL Server 2005. Organizations that are looking for a set of tools for Service Broker deployment, maintenance and administration have now a commercially supported alternative. 

<!--more-->

The  <a href="http://www.quest.com/images/popup.asp?path=/toad-for-sql-server/images/ToadSS30ServiceBroker.jpg&#038;width=1024&#038;height=710" target="_blank">Service Broker Manager</a> tools for Toad are available on all versions (Professional, Xpert and Dev suite), according to this <a href="http://www.quest.com/toad-for-sql-server/configurations-checklist.aspx" target="_blank">comparison chart</a>. After you connect to a SQL Server instance from the Connection Manager you get the option to launch the Service Broker Manager in the tools menu:

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/servicebroker-menu.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/servicebroker-menu.png' alt="Service Broker Manager in Tools menu" width="450" title="Click on the image for a full size view" /></a>
</div>

The Service Broker Manager has a typical Object Explorer style tool, with a hierarchical tree view of all databases that are enabled for Service Broker and all broker related objects in each database.

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/servicebrokermanager-full.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/servicebrokermanager-full.png' alt="Service Broker Manager Full view" width="450" title="Click on the image for a full size view" /></a>
</div>

One thing I like about the Toad model of the Object Explorer is that it has a brief description for each object type definition. If you are new to Service Broker and have never encountered before a Queue, you can click the Description tab and read the definition of a Queue, similar to what you would find in Books On Line on the same subject. The description tab also contains quick action links for the context of the object type, like a &#8216;Create Queue&#8217; link. While the quick link action idea is nice, I find a bit confusing its placement on the &#8216;Definition&#8217; tab.

As expected the context menu of each object in this tree view has options to view properties, create, modify and drop each object type. Interestingly the hierarchy of objects includes all the expected Service Broker types like services, queue and contract, but also object that are not &#8216;owned&#8217; by Service Broker, but are indeed heavily used by it, like certificates and XML Schemas. I found this very interesting, that Quest has considered putting the object where they are used and needed as opposed to where they &#8216;officially&#8217; belong in the Microsoft blessed taxonomy of objects in the database. For instance the SQL Server Management Studio shows the Certificates in the &#8216;Security&#8217; node because they are a Security feature and the XML Schemas in the Programmability/Types/XML Schema Collections node, because they are an XML data type feature. Just like supermarkets place tomato sauce right next to pasta on the shelves, Quest has placed the objects where they are needed, not in the parent node of the team that owns the feature. Of course there are other tools in Toad like the Database Browser tool that are showing the Certificates where they &#8216;properly&#8217; belong in the Security tab, since Certificates truly are a &#8216;Security&#8217; feature and are used in code signing, encryption and other strictly security related roles.

The Service Broker Manager has functionalities that cover all the broker needs, including configuring endpoints, enabling and disabling broker in the database and even setting the trustworthy bit on or off on the database:

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/databaseproperties.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/databaseproperties.png' alt="Service Broker Database properties" title="Click on the image for a full size view" width="450" /></a>
</div>

Another nice feature of the Service Broker Manager is the fact that it shows conversation endpoints, messages in queues, messages for each conversation and, last but not least, the database transmission queue. The current SQL Server Management Studio for SQL Server 2005 doesn&#8217;t have anything similar. The SQL Server 2008 SSMS has the Broker Statistics Report that also shows the conversations and messages as well as the transmission queue. But the Toad one goes not one, but two steps further. For one it allows you to actually Edit conversations:

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/conversationproperties.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/conversationproperties.png' alt="Conversation Properties" title="Click on the image for a full size view" width="450" /></a>
</div>

The only &#8216;editable&#8217; properties on a conversation are it&#8217;s conversation group and its timer. I reckon I was very surprised to see these two properties as editable. I was so used to see them as T-SQL statements (<a href="http://msdn.microsoft.com/en-us/library/ms174987.aspx" target="_blank">MOVE CONVERSATION</a> and <a href="http://msdn.microsoft.com/en-us/library/ms187804.aspx" target="_blank">BEGIN CONVERSATION TIMER</a>) that I have never thought about them as means to &#8216;edit&#8217; a conversation property. Is true that they may appear so for an administrator looking at a conversation, but they serve clear application programming roles related to <a href="http://msdn2.microsoft.com/en-us/library/ms171615.aspx" target="_blank">correlated messages locking</a> (the MOVE CONVERSATION) or to retry mechanisms and time outs (the BEGIN CONVERSATION TIMER).

And the second &#8216;further step&#8217; Toad offers in regard to conversations is that it actually has options to begin and end conversations and even send a message:

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/send.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/send.png' alt="Send Message" width="150" title="Click on the image for a full size view" /></a>
</div>

Now all these would make a nice tool for administering Service Broker, with some richer functionality that SSMS offers. But the I saved the best features for last.

## The Wizards

What makes the Toad SSB tool **really** useful are its wizards: 

  * Transport Security Configuration Wizard
  * Service Broker Application Wizard
  * Server Event Notification Application Wizard
  * Database Event Notification Application Wizard

The Transport Security Configuration Wizard will configure endpoints and transport security between two SQL Server instances, including the exchange of certificates in case of certificate based security. It allows the user to select the desired authentication mode (certificates and/or Windows) and it will take the user step by step through all the necessary settings involved, if needed, including configuring the database master key password, certificate creation etc. The Transport Security Configuration Wizard has to be run only once between any two SQL Server instances.

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/transportsecurity.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/transportsecurity.png' alt="Transport Security Wizard" width="250" title="Click on the image for a full size view" /></a>
</div>

The Service Broker Application Wizard will take you step by step to setup a complete SSB application between two services. It allows you to create or choose an existing service for initiator and target, create or choose an existing contract and it will configure routing and security between the two services. If it detects that the two services are located on separate SQL Server instances it will launch the Transport Security Configuration Wizard if needed in order to configure the transport security separately. Not only that, but if you create a new queue it will also create a stored procedure for activation, based on a template with a message RECEIVE loop that handles the system Error and EndDialog message types and has a placeholder where you have to put the logic to handle your own contract messages.

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/application.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/application.png' alt="Service Broker Application Wizard" width="250" title="Click on the image for a full size view" /></a>
</div>

The two Event Notification Wizards are for configuring a service queue&nbsp; that can handle an event notification. The Wizard will let you choose the event notification scope and type, create an event notification subscription and can even automatically create the service and queue for you, with names derived from the subscription name.

<div class="post-image">
  <a href='http://test.rusanu.com/wp-content/uploads/2008/05/eventnotifications.png' target="_blank"><img src='http://test.rusanu.com/wp-content/uploads/2008/05/eventnotifications.png' alt="Event Notifications Application Wizard" width="250" title="Click on the image for a full size view" /></a>
</div>

## Odds and Ends

I consider the Toad Service Broker Manager a much better alternative than SSMS for Service Broker. This should come as no surprise, after all the SSMS tools come for free with a SQL Server purchase and Quest has to make better tools if is to stay in business. I reckon though I was pleasantly surprised by the depth of coverage Toad&#8217;s has to offer for SSB. The Wizards are awesome and exactly what you need to get yourself started on SSB.

But there are some odds and ends, some rough edges in the Service Broker Manager too. Some examples would be that the Service Broker Application Wizard forces you to choose a stored procedure for activation, omitting cases like an external application that might read the queue. Also the Service Broker Application wizard **_only_** lets you configure public security between services and cannot configure secure conversations. How ironic that the Service Listing Manager only lets you configure secure conversations and cannot deal with public security 🙂 _[Correction 5/22: there is a separate [Dialog Security Wizard](http://rusanu.com/2008/05/22/dialog-security-configuration-wizard-in-toad) that allows you to configure both anonymous and full security]_

There are also some problems I encountered and had to work my way around. For instance the Transport Security Wizard uses the computer administrative share (MACHINE\Admin$) to exchange certificates, which requires the SQL Server instance to be able to access this share. Most times certificates are deployed exactly because the two machines _cannot_ authenticate using Windows, so access to the administrative share is unlikely.

I must also add that this post is not an endorsement of any sort nor payed advertising. I simply heard about the tools and downloaded the trial version to test them, just like any of you can, but I am very pleased with what I see and I had to share with my readers.