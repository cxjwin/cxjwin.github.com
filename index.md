---
layout: index
title: Smart Cai's Blog 
tagline: iOS developer
---

### About Author

<br>Name   : 	Smart Cai</br>
<br>Email  :	[88cxjwin@gmail.com][email]</br>
<br>Github :	[cxjwin.github.com][github]</br>

[email]:"88cxjwin@gmail.com"
[github]:"cxjwin@github.com"

### Blog List

<ul class="posts">
  {% for post in site.posts %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>

