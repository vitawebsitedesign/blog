---
title: Blog
layout: default
---
{% for post in site.posts %}
	{{ post.title }}
	{{ post.date | date: "%b %d, %Y" }}
	{{ post.content }}
{% endfor %}
