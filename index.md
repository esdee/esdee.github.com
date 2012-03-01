---
layout: page
title: Shashy's blog
---
{% include JB/setup %}

<h3>Clojure, Ruby and so on</h3>
<div class="index-page-posts">
  <ul class="posts">
    {% for post in site.posts %}
      <li><span>{{ post.date | date_to_string }}</span> &raquo;
      <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
</div>
