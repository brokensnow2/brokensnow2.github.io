---
title: Welcome
layout: default
---

# Welcome to My Blog 👋

Hi, I'm **dx x** — I write about robotics, AI, and vision-language navigation.

📖 [About me](about)

---

📝 Recent Posts:

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
<p>No posts yet — stay tuned!</p>
{% endif %}
