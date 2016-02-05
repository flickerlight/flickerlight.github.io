---
layout: page
title: Other 
---

## Blog Posts
{% for post in site.posts %}
{%for pc in post.categories %}
{%if pc == "other" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endif %}
{% endfor %}
{% endfor %}
