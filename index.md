---
title: Blog
layout: default
---
{% for post in site.posts %}
	<article class="{% if forloop.first %}first{% elsif forloop.last %}last{% else %}middle{% endif %}">
		{{ post.url }}
		{{ post.title }}
		{{ post.date | date: "%b %d, %Y" }}
		{{ post.content }}
	</article>

	{% if forloop.last %}
	{% else %}
	<hr />
	{% endif %}
{% endfor %}
