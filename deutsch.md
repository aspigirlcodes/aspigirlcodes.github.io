---
layout: page
title: Deutsch
order: 10
---
<h1>Beiträge auf Deutsch</h1>
<div id="posts">

{% for post in site.categories.Deutsch %}

  <div class="post">
  	<h2><a href="{% if site.baseurl == "/" %}{{ post.url }}{% else %}{{ post.url | prepend: site.baseurl }}{% endif %}">{%if post.header %}{{ post.header }}{% else %}{{ post.title }}{% endif %}</a></h2>
    <time datetime="{{ post.date | date_to_xmlschema }}" class="by-line"> <i>{{ post.date | date_to_string }}</i> </time>
  	<p>{{ post.content | strip_html | truncatewords:50 }}</p>
  </div>

  {% endfor %}

</div>
