---
layout: page
permalink: /work-tags/
title: Technical Posts
---


<div id="archives">
  {% assign tag_set = nil %}
  {% for post in site.categories["Work"] %}
    {% for tag in post.tags %}
      {% if tag_set == nil %}
        {% assign tag_set = tag %}
      {% else %}
        {% assign tag_set = tag_set | append: "," | append: tag %}
      {% endif %}
    {% endfor %}
    <article class="archive-item">
      <h4><a href="{{ site.baseurl }}{{ post.url }}">{% if post.title and post.title != "" %}{{post.title}}{% else %}{{post.excerpt |strip_html}}{%endif%}</a></h4>
    </article>
  {% endfor %}
  {% assign tag_array = tag_set | split: "," | uniq %}
  <div class="post-tags">
    <h1>Tags</h1>
      {% for tag in tag_array %}
        <a href="{{site.baseurl}}/tags/#{{tag|slugize}}">{{tag}}</a>
        {% unless forloop.last %}&nbsp;{% endunless %}
      {% endfor %}
  </div>
</div>
