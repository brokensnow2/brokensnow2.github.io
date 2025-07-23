---
title: Welcome
layout: default
---

# Welcome to My Blog 👋

Hi, I'm **dx x** — I write about robotics, AI, and vision-language navigation.

📖 [About me](about)

📝 Recent Posts:
<ul>
  {% for post in site.posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> — {{ post.date | date: "%Y-%m-%d" }}</li>
  {% endfor %}
</ul>
