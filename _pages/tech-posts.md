---
permalink: /tech-posts/
title: "Tech Posts"
---

{% for tag in site.tags %}
  {% if tag[0] == "tech" %}
    <ul>
      {% for post in tag[1] %}
        <li><a href="{{ post.url }}">{{ post.title }}</a></li>
      {% endfor %}
    </ul>
  {% endif %}
{% endfor %}