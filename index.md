---
title: Blog
layout: default
---
## x
{% for post in site.posts %}
  <article class="{% if forloop.first %}first{% elsif forloop.last %}last{% else %}middle{% endif %}">

	</article>
	{% if forloop.last %}
	{% else %}
	---
	{% endif %}
{% endfor %}
