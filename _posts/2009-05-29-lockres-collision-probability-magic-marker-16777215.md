---
id: 410
title: '%%lockres%% collision probability magic marker: 16,777,215'
date: 2009-05-29T00:41:08+00:00
author: remus
layout: post
guid: http://rusanu.com/?p=410
permalink: /2009/05/29/lockres-collision-probability-magic-marker-16777215/
categories:
  - Troubleshooting
tags:
  - birthday attack
  - collision
  - deadlock
  - hash
  - sql
---
<a href="http://blogs.conchango.com/jamesrowlandjones" target="_blank">@jrowlandjones</a> blogged about a <a href="http://blogs.conchango.com/jamesrowlandjones/archive/2009/05/28/the-curious-case-of-the-dubious-deadlock-and-the-not-so-logical-lock.aspx" target="_blank">dubious deadlock case.</a> I recommend this article as is correct and presents a somewhat esoteric case of deadlock: the resource hash collision. The lock manager in SQL Server doesn&#8217;t know what it locks, it just locks &#8216;resources&#8217; (basically **strings**). It is the job of higher level components like the the access methods of the storage engine to present the &#8216;resource&#8217; to the lock manager and ask for the desired lock. When locking rows in a heap or a b-tree the storage engine will synthesize a &#8216;resource&#8217; from the record identifier. Since these resources have a limited length, the storage engine has to reduce the effective length of a key to the maximum length is allowed to present to the lock manager, and that means that the record&#8217;s key will be reduced to 6 bytes. This is achieved by hashing the key into a 6 byte hash value. Nothing spectacular here.

But if you have a table with a key of length 50 bytes and its reduced to 6 bytes, you may hit a collision. So how likely is this to happen?

On 6 bytes there are 281,474,976,710,656 distinct possible values. Its a pretty big number? Not that big actually. If we meet at a party and I say &#8216;I bet somebody in the room shares your birthday&#8217; would you bet against me? You probably should 🙂 What if I change my question to &#8216;I bet there are two people in this room that share the birthday!&#8217;? Now I will probably take your money. This is called a &#8216;<a href="http://en.wikipedia.org/wiki/Birthday_attack" target="_blank">meet-in-the-middle</a>&#8216; attack in cryptography and basically it says that you get a 50% collision probability at half the hash length. So the SQL %%lockres%% hash will produce two records with same hash, with a 50% probability, out of the table, any table, of only 16,777,215 record. That suddenly doesn&#8217;t look like a cosmic constant, does it? And keep in mind that this is the absolutely maximum value, where the key has a perfectly random distribution. In reality keys are not that random after all. Take Jame&#8217;s example: a datetime (8 bytes), a country code (1 byte), a group code (2 bytes) and a random id (4 bytes). From these 15 bytes quite a few are actually constant: eg. every date between 2000 and 2010 has the first 4 bytes identical (0x00) and the 5th byte only has two possible values (0x08 or 0x09). If from the other codes (country, group) we use only 50% of the possible values, then in effect we use, generously, just 10 bytes of the 15 bytes of the key. This means the table has a 50% collision probability at only about 11 million records. considering that he was doing a &#8216;paltry&#8217; 900 million records upload, no wonder he got collisions&#8230;