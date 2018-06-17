---
layout: default
title: "包丁一本さらしに巻いて"
---
{% for post in site.posts %}
  <section>
    <h3><a class="post-title" href="{{ post.url }}">{{ post.title }}</a></h3>
    <p>{{ post.date | date: "%Y.%m.%d" }}</p>
    <p>{{ post.excerpt }}</p>
  </section>
{% endfor %}
<script>
  (function(i,s,o,g,r,a,m){i['GoogleAnalyticsObject']=r;i[r]=i[r]||function(){
  (i[r].q=i[r].q||[]).push(arguments)},i[r].l=1*new Date();a=s.createElement(o),
  m=s.getElementsByTagName(o)[0];a.async=1;a.src=g;m.parentNode.insertBefore(a,m)
  })(window,document,'script','//www.google-analytics.com/analytics.js','ga');

  ga('create', 'UA-50431971-1', 'akirachiku.com');
  ga('send', 'pageview');
</script>
