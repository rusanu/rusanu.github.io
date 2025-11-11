---
id: 42
title: Blog
date: 2007-10-26T16:37:51+00:00
author: remus
layout: page
guid: /blog/
---
<div class="posts">
{% for post in site.posts %}
<div class="post" id="post-{{ post.id }}">
<div class="post-title">
<h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
<small>{{ post.date | date: "%B %-d, %Y" }}</small>
</div>
</div>
{% endfor %}
</div>
