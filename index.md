---
title: Blog
layout: default
---

<section>
	<h1>Memento</h1>
	<h4>Software Engineer blog</h4>
</section>
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

<script type="text/x-mathjax-config">
MathJax.Hub.Config({
    TeX: {
      equationNumbers: {
        autoNumber: "AMS"
      }
    },
    tex2jax: {
    inlineMath: [ ['$', '$'] ],
    displayMath: [ ['$$', '$$'] ],
    processEscapes: true,
  }
});
MathJax.Hub.Register.MessageHook("Math Processing Error",function (message) {
	  alert("Math Processing Error: "+message[1]);
	});
MathJax.Hub.Register.MessageHook("TeX Jax - parse error",function (message) {
	  alert("Math Processing Error: "+message[1]);
	});
</script>
<script type="text/javascript" async
  src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-MML-AM_CHTML">
</script>
