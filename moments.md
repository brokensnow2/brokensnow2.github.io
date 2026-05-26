---
layout: default
title: Moments
permalink: /moments/
---

# Moments

Small updates from music, videos, photos, and other fragments that do not need to become technical posts.

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
