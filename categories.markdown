---
layout: page
permalink: /categories/
title: Categories
---
<div class="home">
{%- for category in site.categories -%}
  <h2 class="post-list-heading">{{ category[0] }}</h2>
  <ul class="post-list">
    {%- for post in category[1] -%}
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
      <li>
        <span class="post-meta">
          {{ post.date | date: date_format }}
        </span>
        <a class="post-link" href="{{ post.url | relative_url }}">
          {{ post.title | escape }}
        </a>
      </li>
    {%- endfor -%}
  </ul>
{%- endfor -%}
</div>