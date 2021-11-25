---
id: 2120
title: Using Hive local mode with YARN
date: 2013-11-19T04:35:26+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/11/19/2089-revision-5/
permalink: /2013/11/19/2089-revision-5/
---
Most examples on how to run Hive in local mode mention using <tt>SET mapred.job.tracker=local</tt>. But with YARN this setting no longer has the desired effect, Hive queries are still executed with the M/R framework. Stepping through the [Hadoop 23 shims](https://github.com/apache/hive/blob/trunk/shims/0.23/src/main/java/org/apache/hadoop/hive/shims/Hadoop23Shims.java) reveals that the decision to split into local mode is driven by different configuration setting:

    
      @Override
      public boolean isLocalMode(Configuration conf) {
        return "local".equals(conf.get("mapreduce.framework.name"));
      }
    

Running Hive in localmode is important in debugging, is much easier to attach a remote to the local spawned Java process than to a real cluster, even a local one-node cluster. Thejas has a this short note about [running hive in local mode](http://hadoop-pig-hive-thejas.blogspot.ie/2013/04/running-hive-in-local-mode.html), but following the advise there does not work on YARN clusters like the more recent [Hortonworks sandboxes](http://hortonworks.com/products/hortonworks-sandbox/#install).