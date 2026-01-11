---
layout: home
title: "Posts en Blue Team"
---

<h1>Posts en categoría Blue Team</h1>
<ul>
  {% assign posts = site.categories['blue-team'] %}
  {% for post in posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> — <small>{{ post.date | date: "%d %b %Y" }}</small></li>
  {% endfor %}
</ul>
