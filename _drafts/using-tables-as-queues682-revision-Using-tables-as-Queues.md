---
id: 683
title: Using tables as Queues
date: 2010-03-04T21:52:36+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/04/682-revision/
permalink: /2010/03/04/682-revision/
---
A very common question asked on all programming forums is how to implement queues based on database tables. This is not a trivial question actually. Implementing a queue backed by a table is notoriously difficult, error prone and susceptible to deadlocks. Because queues are usually needed as a link between various processing stages in a workflow they operate in highly concurrent environments where multiple processes enqueue rows into the table while multiple processes attempt to dequeue these rows. This concurency poses both correctness problems and scalability and performance problems.