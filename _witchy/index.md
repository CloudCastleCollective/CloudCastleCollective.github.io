---
layout: page
title: Witchy
permalink: /witchy/
entries_layout: grid
show_excerpts: true
---

{% assign entries = site.categories.witchy %}
<div class="entries-{{ page.entries_layout | default: 'list' }}">
{% for entry in entries %}
  {% include entries-grid.html %}
{% endfor %}
</div>
