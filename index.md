---
title: Blog
layout: default
---

<section>
	<h1>All posts:</h1>
	<ul>
		{% for post in site.posts %}
			<li>
				<a href="{{ post.url }}">{{ post.title }}</a>
			</li>
		{% endfor %}
	</ul>
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
