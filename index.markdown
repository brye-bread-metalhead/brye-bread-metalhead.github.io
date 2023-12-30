---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
---

<h2>Latest Blog Posts</h2>
<ul>
{% for post in site.posts limit:3 %}
  <li>
    <a href="{{ post.url }}">{{ post.title }}</a>
    <p>{{ post.date | date: "%B %d, %Y" }}</p>
  </li>
{% endfor %}
</ul>

