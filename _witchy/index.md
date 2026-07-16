---
layout: page
title: Witchy
permalink: /witchy/
entries_layout: grid
show_excerpts: false
---

{% assign entries = site.categories.witchy %}
<div class="entries-grid">
{% for entry in entries %}
  {% include entry.html %}
{% endfor %}
</div>
