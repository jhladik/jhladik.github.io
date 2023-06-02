---
layout: page
permalink: /categories/
title: Categories
date: 2023-05-19 10:58:00 -0600
---
<div class="home">
{%- for category in site.categories -%}
  <h2>{{ category[0] }}</h2>
  <ul class="category-post-list">
    {%- for post in category[1] -%}
      {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
      <li>
        <a href="{{ post.url | relative_url }}">
          {{ post.title | escape }}
        </a>
        <span class="category-post-meta">
          {{ post.date | date: date_format }}
        </span>
      </li>
    {%- endfor -%}
  </ul>
{%- endfor -%}
</div>