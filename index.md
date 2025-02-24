---
layout: page
title: RPblog Index
---

Just a small blog to gather mostly IT topics.

<h2 class="post-list-heading">Posts</h2>

<ul class="post-list">
 {% for post in site.posts %}
  <li>
   <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
   <h3><a class="post-link" href="{{ post.url }}">{{ post.title }}</a></h3>
  </li>

{% endfor %}
</ul>
