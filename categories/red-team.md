---
layout: home
title: "Posts en Red Team"
---

<h1>Posts en categoría Blue Team</h1>
<ul>
  {% assign posts = site.categories['red-team'] %}
  {% for post in posts %}
    <li><a href="{{ post.url }}">{{ post.title }}</a> — <small>{{ post.date | date: "%d %b %Y" }}</small></li>
  {% endfor %}
</ul>
