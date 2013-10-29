---
layout: index
title: Smart Cai's Blog 
tagline: iOS developer
---

### About Author

Name : Smart Cai
Email : 88cxjwin@gmail.com
Github : cxjwin.github.com

### Blog List

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

