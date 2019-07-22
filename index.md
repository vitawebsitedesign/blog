---
title: Blog
layout: default
---

<section>
  {% for post in site.posts %}
	<article>
		<h1>{{ post.title }}</h1>
		<p>
			{{ post.content }}
		</p>
	</article>
	<br />
	<br />
	<hr />
	<br />
	<br />
  {% endfor %}
</section>
