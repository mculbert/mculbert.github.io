---
title: Index
global: true
---

# By Category
{% for cat_pair in site.categories %}
  {% assign cats = cat_pair | first | append: "," | append: cats %}
{% endfor %}
{% assign cats = cats | split: "," | sort_natural %}
{% for cat in cats %}
* [{{ cat | capitalize }}]({{ cat | remove: ' ' }})
{% endfor %}
