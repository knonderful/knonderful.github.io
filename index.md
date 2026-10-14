---
layout: default
---

# Welcome

This is pretty bare for the time being.

# Recent Posts

<ul>
{% for post in site.posts limit:5 %}
  <li>
    <a href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <small>({{ post.date | date: "%Y-%m-%d" }})</small>
  </li>
{% endfor %}
</ul>

[View all posts]({{ "/blog/" | relative_url }})
