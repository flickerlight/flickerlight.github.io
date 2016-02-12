---
layout: page
title: Other
---

## Blog Posts
{% for post in site.posts %}
{%if post.category == "Other" %}
  * {{ post.date | date_to_string }} &raquo; [ {{ post.title }} ]({{ post.url }})
{% endif %}
{% endfor %}
