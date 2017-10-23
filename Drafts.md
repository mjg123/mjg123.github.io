---
layout: page
title: wip
permalink: /drafts/
---

{% assign date_format = site.minima.date_format | default: "%b %-d, %Y" %}

<div class="tagblock">
<img src="{{ "assets/tag_purple.png" | relative_url }}" style="width: 15px"/>
<strong>drafts</strong>

 <ul>
   {% for post in site.posts %}
      {% if post.tags contains 'draft' %} 
         
        <li>
			<a href="{{ post.url }}">{{ post.title }}</a>
			<ul class="tag-list">
				{% for tag in post.tags %}
				<li>{{tag}}</li>
				{%endfor%}
			</ul>
        </li>

      {% endif %}
  {% endfor %}
</ul>

</div>

