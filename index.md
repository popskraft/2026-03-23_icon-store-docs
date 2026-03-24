---
layout: default
title: "Shared Project Documents"
description: "A small link-only portal for the project documents that live in `_docs/`."
---

{% assign docs = site.pages | where: "doc_entry", true | sort: "nav_order" %}

This site is meant for direct sharing by URL. It is intentionally configured with `noindex` meta tags and a restrictive `robots.txt`, but anyone who has a page link can still open it.

## Available Documents

<ul class="doc-list">
{% for doc in docs %}
  <li>
    <a href="{{ doc.url | relative_url }}">{{ doc.title }}</a>
    {% if doc.description %}
    <p>{{ doc.description }}</p>
    {% endif %}
  </li>
{% endfor %}
</ul>
