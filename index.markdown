<!-- ---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
--- -->
<!-- ---
layout: default
title: Blog Posts
--- -->

{% for post in site.posts %}
  <article>
    <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
    <p>{{ post.excerpt }}</p>
    <small>{{ post.date | date: "%B %d, %Y" }}</small>
  </article>
{% endfor %}