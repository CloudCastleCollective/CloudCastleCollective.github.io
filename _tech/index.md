---
layout: page
title: Tech
permalink: /tech/
entries_layout: grid
show_excerpts: true
---

{% assign entries = site.categories.tech %}
<div class="entries-grid">
{% for entry in entries %}
  {% include entry.html %}
{% endfor %}
</div>
