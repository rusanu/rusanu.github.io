---
id: 685
title: Using tables as Queues
date: 2010-03-06T11:50:29+00:00
author: remus
layout: revision
guid: http://rusanu.com/2010/03/06/682-revision-3/
permalink: /2010/03/06/682-revision-3/
---
A very common question asked on all programming forums is how to implement queues based on database tables. This is not a trivial question actually. Implementing a queue backed by a table is notoriously difficult, error prone and susceptible to deadlocks. Because queues are usually needed as a link between various processing stages in a workflow they operate in highly concurrent environments where multiple processes enqueue rows into the table while multiple processes attempt to dequeue these rows. This concurrency creates correctness, scalability and performance challenges.

## Heap Queue

The simplest queue is a heap: producers can equeue into the heap and