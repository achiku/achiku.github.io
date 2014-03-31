---
layout: default
title: "包丁一本さらしに巻いて"
---
{% for post in site.posts %}
  <section>
    <h2><a class="post-title" href="{{ post.url }}">{{ post.title }}</a></h2>
    <p>{{ post.date | date: "%Y.%m.%d" }}</p>
    <p>{{ post.excerpt }}</p>
  </section>
{% endfor %}
