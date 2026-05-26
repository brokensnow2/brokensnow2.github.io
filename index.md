---
title: Home
layout: default
---

# Hi, I'm dx x

I'm a graduate researcher working on **Vision-and-Language Navigation**, **Embodied AI**, **Robotics**, **3D Perception**, and small creative systems.

<div class="bio-links">
  <a href="/about/">About</a>
  <a href="/projects/">Projects & Publications</a>
  <a href="/posts/">Recent Posts</a>
  <a href="/moments/">Moments</a>
  <a href="mailto:brokensnow2@gmail.com">Email</a>
  <a href="https://github.com/brokensnow2">GitHub</a>
</div>

---

## Research Interests

<div class="interests-grid">
  <div class="interest-card">
    <span class="interest-icon">🧭</span>
    <strong>VLN</strong>
    <p>Vision-and-Language Navigation: agents that follow natural language instructions in 3D environments.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🤖</span>
    <strong>Embodied AI</strong>
    <p>Building agents that perceive, reason, and act in physical or simulated worlds.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🧊</span>
    <strong>3D Perception</strong>
    <p>Understanding geometry and semantics from point clouds, depth maps, and multi-view imagery.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🔧</span>
    <strong>Embedded Development</strong>
    <p>Low-level systems, microcontrollers, sensors, and hardware-software experiments.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🎧</span>
    <strong>Music</strong>
    <p>Songwriting, sound design, and the emotional side of tools, rhythm, and texture.</p>
  </div>
  <div class="interest-card">
    <span class="interest-icon">🎮</span>
    <strong>Game Dev</strong>
    <p>Side interest in procedural generation and real-time rendering as a creative outlet.</p>
  </div>
</div>

---

## Recent Posts

{% if site.posts.size > 0 %}
<ul class="post-list">
  {% for post in site.posts limit:3 %}
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
<p><a class="more-link" href="/posts/">View all posts</a></p>
{% else %}
<p>No posts yet. Stay tuned!</p>
{% endif %}
