---
title: Blog
layout: default
---
{% for post in site.posts %}
	<article>
		<a href="{{ post.url }}">
			<h1>
				{{ post.title }}
			</h1>
		</a>
		<div>
			<i>
				{{ post.date | date: "%b %d, %Y" }}
			</i>
		</div>
		<div>
			{{ post.content }}
		</div>
	</article>
{% endfor %}
