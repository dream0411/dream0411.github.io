---
layout: default
title: Debug Me
---

{% for post in site.posts %}
* [{{ post.title }}]({{ site.baseurl }}{{ post.url }})		{{ post.date | date_to_string }}
{% endfor %}
