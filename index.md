---
title: Blog
layout: default
---

<section>
	<small>
		<div>All posts:</div>
		<ul>
			{% for post in site.posts %}
				<li>{{ post.title }}</li>
			{% endfor %}
		</ul>
	</small>
	<hr />
</section>
<section>
  {% for post in site.posts %}
	<article>
		<h1>{{ post.title }}</h1>
		<p><i>{{ post.date | date_to_long_string: "ordinal" }} | {{ post.author }}</i></p>
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
