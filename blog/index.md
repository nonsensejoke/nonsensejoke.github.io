---
layout: default
title: Nonsense Jokes' Blog
---

# 博客文章列表

{%- comment -%}
分组内容在 _data/sections.yml 里手动维护，本文件一般不用改。
{%- endcomment -%}

{%- assign assigned = "||" -%}
{%- for s in site.data.sections -%}
  {%- for key in s.posts -%}
    {%- assign assigned = assigned | append: key | append: "||" -%}
  {%- endfor -%}
{%- endfor -%}

<p>
{%- for s in site.data.sections %}
  <a href="#{{ s.id }}">{{ s.name }}</a>{% unless forloop.last %} · {% endunless %}
{%- endfor %}
</p>

{% for s in site.data.sections %}
<h2 id="{{ s.id }}">{{ s.name }} <small>（{{ s.posts.size }}）</small></h2>
<ul>
{%- for key in s.posts -%}
  {%- for post in site.posts -%}
    {%- assign stem = post.path | remove_first: "_posts/" | remove: ".md" -%}
    {%- if stem == key %}
  <li><a href="{{ post.url }}">{{ post.title }}</a> <small>{{ post.date | date: "%Y-%m-%d" }}</small></li>
    {%- endif -%}
  {%- endfor -%}
{%- endfor %}
</ul>
{% endfor %}

{%- assign unsorted = 0 -%}
{%- for post in site.posts -%}
  {%- assign stem = post.path | remove_first: "_posts/" | remove: ".md" -%}
  {%- assign needle = "||" | append: stem | append: "||" -%}
  {%- unless assigned contains needle -%}
    {%- assign unsorted = unsorted | plus: 1 -%}
  {%- endunless -%}
{%- endfor -%}

{% if unsorted > 0 %}
<h2 id="unsorted">未分类 <small>（{{ unsorted }}）</small></h2>
<p><small>下面的文章还没有写进 <code>_data/sections.yml</code>。</small></p>
<ul>
{%- for post in site.posts -%}
  {%- assign stem = post.path | remove_first: "_posts/" | remove: ".md" -%}
  {%- assign needle = "||" | append: stem | append: "||" -%}
  {%- unless assigned contains needle %}
  <li><a href="{{ post.url }}">{{ post.title }}</a> <small>{{ post.date | date: "%Y-%m-%d" }}</small></li>
  {%- endunless -%}
{%- endfor %}
</ul>
{% endif %}
