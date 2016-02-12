---
layout: page
title: Healthcare 
---

## Blog Posts
{% for post in site.posts %}
{%if post.category == "Healthcare" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endif %}
{% endfor %}
