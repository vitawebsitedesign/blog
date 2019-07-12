---
title: Blog
layout: default
---
## x
{% for post in site.posts %}
  <article class="{% if forloop.first %}first{% elsif forloop.last %}last{% else %}middle{% endif %}">
	{{ post }}
	{% if forloop.last %}{% else %}<div class="separater"></div>{% endif %}
{% endfor %}
