---
layout: default
permalink: /blog/
title: blog
nav: true
nav_order: 3
pagination:
  enabled: true
  collection: posts
  permalink: /page/:num/
  per_page: 5
  sort_field: date
  sort_reverse: true
  trail:
    before: 1 # The number of links before the current page
    after: 3 # The number of links after the current page
---

<div class="post blog-index">

{% assign all_posts = site.posts %}
{% assign series_names = all_posts | map: "series" | compact | uniq %}
{% assign is_first_page = true %}
{% if paginator and paginator.page > 1 %}{% assign is_first_page = false %}{% endif %}

{% if is_first_page %}
<header class="blog-hero">
  <p class="blog-eyebrow">Blog</p>
  <h1 class="blog-title">Notes on <span class="grad">my research</span></h1>
  <p class="blog-count"><strong>{{ all_posts.size }}</strong> posts · <strong>{{ series_names.size }}</strong> series</p>
</header>

{% if series_names.size > 0 %}
<section class="blog-series">
  {% for name in series_names %}
    {% assign sp = all_posts | where: "series", name | sort: "date" %}
    {% assign first = sp | first %}
    {% assign last = sp | last %}
    <a class="series-card" href="{{ first.url | relative_url }}">
      <span class="series-card-label">Series · {{ sp.size }} posts</span>
      <span class="series-card-name">{{ name }}</span>
      <ol>
        {% for p in sp %}<li>{{ p.title }}</li>{% endfor %}
      </ol>
      <span class="series-card-cta">Start from part 1 →</span>
    </a>
  {% endfor %}
</section>
{% endif %}
{% endif %}

{% if site.display_tags and site.display_tags.size > 0 or site.display_categories and site.display_categories.size > 0 %}
<div class="tag-category-list">
  <ul class="p-0 m-0">
    {% for tag in site.display_tags %}
      <li><a href="{{ tag | slugify | prepend: '/blog/tag/' | relative_url }}">#{{ tag }}</a></li>
    {% endfor %}
    {% for category in site.display_categories %}
      <li class="category-chip"><a href="{{ category | slugify | prepend: '/blog/category/' | relative_url }}">{{ category | replace: '-', ' ' }}</a></li>
    {% endfor %}
  </ul>
</div>
{% endif %}

{% if page.pagination.enabled %}
  {% assign postlist = paginator.posts %}
{% else %}
  {% assign postlist = site.posts %}
{% endif %}

<div class="blog-list">
{% for post in postlist %}
  {% if post.redirect == blank %}
    {% assign post_url = post.url | relative_url %}
  {% elsif post.redirect contains '://' %}
    {% assign post_url = post.redirect %}
  {% else %}
    {% assign post_url = post.redirect | relative_url %}
  {% endif %}
  <article class="blog-card">
    <div class="blog-card-meta">
      {% if post.series %}<span class="blog-card-series">{{ post.series }}</span>{% endif %}
      {% for category in post.categories %}<span class="blog-card-cat">{{ category | replace: '-', ' ' }}</span>{% endfor %}
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: '%b %-d, %Y' }}</time>
    </div>
    <h3><a class="post-title stretched" href="{{ post_url }}">{{ post.title }}</a></h3>
    {% if post.description %}<p class="blog-card-desc">{{ post.description }}</p>{% endif %}
    {% if post.tags.size > 0 %}
    <div class="blog-card-tags">
      {% for tag in post.tags %}
        <a href="{{ tag | slugify | prepend: '/blog/tag/' | relative_url }}">#{{ tag }}</a>
      {% endfor %}
    </div>
    {% endif %}
  </article>
{% endfor %}
</div>

{% if page.pagination.enabled %}
{% include pagination.liquid %}
{% endif %}

</div>
