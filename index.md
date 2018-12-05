---
layout: index
title:  bbw's blog
---

{% for post in site.posts %}
- ### [{{ post.title }}]({{ post.url }})
<i class="fa fa-calendar "></i> <time >{{ post.date | date: '%Y-%m-%d' }}</time>
<i class="icon-folder-open"></i> [{{post.category}}](/categories.html#{{post.category}})
<i class="fa fa-tags"></i>
{% for tag in post.tags %}[{{tag}}](/tags.html#{{tag}}) {% endfor %}

  {{ post.summary }}

  [Read more &raquo;]({{ post.url }})
{% endfor %}
