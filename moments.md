---
layout: default
title: Moments
permalink: /moments/
---

# <span data-en="Moments" data-zh="动态">Moments</span>

<p data-en="Small updates: music, videos, photos, and fragments that do not need to become full technical posts." data-zh="这里放一点近况，比如音乐、视频、照片，或者一些还没整理成文章的小片段。">Small updates: music, videos, photos, and fragments that do not need to become full technical posts.</p>

{% assign moments = site.moments | sort: "date" | reverse %}
{% if moments.size > 0 %}
<div class="moments-grid">
  {% for moment in moments %}
    <article class="moment-card moment-{{ moment.type | default: 'note' }}">
      {% if moment.media %}
        <a href="{{ moment.url | relative_url }}" class="moment-media">
          <img src="{{ moment.media | relative_url }}" alt="{{ moment.title }}">
        </a>
      {% else %}
        <a href="{{ moment.url | relative_url }}" class="moment-media moment-placeholder" aria-label="{{ moment.title }}">
          <span>{{ moment.icon | default: "📝" }}</span>
        </a>
      {% endif %}
      <div class="moment-body">
        <span class="moment-type">{{ moment.icon | default: "📝" }} {{ moment.type | default: "note" }}</span>
        <h2><a href="{{ moment.url | relative_url }}">{{ moment.title }}</a></h2>
        <time datetime="{{ moment.date | date_to_xmlschema }}">{{ moment.date | date: "%Y-%m-%d" }}</time>
        {% if moment.description %}
          <p>{{ moment.description }}</p>
        {% endif %}
      </div>
    </article>
  {% endfor %}
</div>
{% else %}
{% include barren-game.html %}
{% endif %}
