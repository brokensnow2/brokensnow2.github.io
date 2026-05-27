---
layout: default
title: Recent Posts
permalink: /posts/
---

# <span data-en="Recent Posts" data-zh="近期文章">Recent Posts</span>

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts %}
    <li class="post-item">
      <a href="{{ post.url }}">{{ post.title }}</a>
      <span class="post-date">{{ post.date | date: "%Y-%m-%d" }}</span>
      {% if post.description %}
        <p class="post-excerpt">{{ post.description }}</p>
      {% elsif post.excerpt %}
        <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
      {% endif %}
    </li>
  {% endfor %}
</ul>
{% else %}
{% include barren-game.html %}
{% endif %}
