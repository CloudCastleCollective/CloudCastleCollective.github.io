---
layout: page
title: RPGs
permalink: /rpg/
entries_layout: grid
---

{% assign entries = site.categories.rpg %}
<div class="entries-grid">
{% for entry in entries %}
  {% include entry.html %}
{% endfor %}
</div>
