---
layout: page
title: Tech
permalink: /tech/
entries_layout: grid
---

{% assign entries = site.categories.tech %}
<div class="entries-grid">
{% for entry in entries %}
  {% include entry.html %}
{% endfor %}
</div>
