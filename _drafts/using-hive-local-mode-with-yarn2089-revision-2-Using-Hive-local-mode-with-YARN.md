---
id: 2117
title: Using Hive local mode with YARN
date: 2013-11-19T04:29:45+00:00
author: remus
layout: revision
guid: http://rusanu.com/2013/11/19/2089-revision-2/
permalink: /2013/11/19/2089-revision-2/
---
Most examples on how to run Hive in local mode mention using <tt>SET mapred.job.tracker=local</tt>. But with YARN this setting no longer has the desired effect, Hive queries are still executed with the M/R framework. Stepping through the Hadoop 23 shims reveals that the decision to split into local mode are driven by different configuration setting:

    
      @Override
      public boolean isLocalMode(Configuration conf) {
        return "local".equals(conf.get("mapreduce.framework.name"));
      }