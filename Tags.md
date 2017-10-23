---
layout: page
title: what I write about
permalink: /tags/
---

{% assign date_format = site.minima.date_format | default: "%b %-d, %Y" %}

{% assign taglist = site.tags | sort %}
{% for tagitem in taglist %} 

<div class="tagblock" id="{{ tagitem[0] }}">
<img src="{{ "assets/tag_purple.png" | relative_url }}" style="width: 15px"/>
<a href="#{{ tagitem[0] }}">
	<strong>{{ tagitem[0] }}</strong>
</a>

 <ul>
   {% for post in site.posts %} 
      {% if post.tags contains tagitem[0] %} 
         
        <li>
			<a href="{{ post.url }}">{{ post.title }}</a>
			<span class="post-meta">({{ post.date | date: date_format }})</span>
        </li>

      {% endif %}
  {% endfor %}
</ul>

</div>
{% endfor %}
