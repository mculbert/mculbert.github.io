---
title: Structured Data Breadcrumbs in Jekyll
description: >
  Programmatic construction of structured data representing hierarchical
  site organization using pure Liquid tags.
category: technology
tags: [jekyll, "structured data", "liquid templating"]
---

[Breadcrumbs](https://en.wikipedia.org/wiki/Breadcrumb_navigation)
help users navigate hierarchically organized websites by providing a
reminder of the path they took (or conceivably would have taken) from the
homepage to the current page, along with links to jump back to any point
along the path. The [Jekyll Codex](https://jekyllcodex.org/) has a nice
example of how to
[generate breadcrumbs programmatically](https://jekyllcodex.org/without-plugin/breadcrumbs)
in [Jekyll](https://jekyllrb.com/) using only pure
[Liquid](https://shopify.github.io/liquid/) tags by parsing the page's URL.
This method works well when the site's logical structure (the path users
take when navigating from the home page) mirrors the directory structure in
the site's URLs.

The Jekyll Codex implementation provides a human-readable representation of
the breadcrumb, but site owners may be interested in augmenting this with a
[machine-readable representation of the breadcrumb](https://developers.google.com/search/docs/data-types/breadcrumb).
Google uses the
[structured data](https://developers.google.com/search/docs/guides/intro-structured-data)
that sites provide to render enhanced search results that may provide search
users with more information to help them decide which results to visit than
a simple full-text preview might.

The following extends the Jekyll Codex breadcrumb idea with a structured
data representation:

<figure class="wide" markdown="1"><figcaption>Breadcrumb structured data</figcaption>
{% raw %}
~~~liquid
{% assign crumbs = page.url | remove: '/index.html' | split: '/' %}
<script type="application/ld+json">
{ "@context": "https://schema.org", "@type": "BreadcrumbList",
  "itemListElement": [{
    "@type": "ListItem", "position": 1,
    "name": "{{ site.title | default: "Home" }}",
    "item": "{{ site.url }}"
{% for crumb in crumbs offset: 1 %}
  },{
    "@type": "ListItem", "position": {{ forloop.index | plus: 1 }},
    "name": "{% if forloop.last %}{{ page.title }}{% else %}{{ site.data.sections[crumb].name | default: crumb | capitalize }}{% endif %}",
    "item": "{{ site.url }}{% assign crumb_limit = forloop.index %}{% for crumb in crumbs offset: 1 limit: crumb_limit %}/{{ crumb }}{% endfor %}"
{% endfor %}
  }]
}
</script>
~~~
{% endraw %}
</figure>

The Jekyll Codex breadcrumb uses directory names as is as the name for each
element in the breadcrumb (replacing hyphens with spaces). This is fine if
sites use full words for section names in their URLs. For my part, I prefer
to abbreviate section names to keep URLs a little shorter. But, the
breadcrumb should show full names, rather than the URL abbreviations. To
accommodate this, we need a mapping between the abbreviations and their full
names. The implementation above makes use of
[Jekyll data files](https://jekyllrb.com/docs/datafiles/) to supply this
mapping, for example:

<figure markdown="1"><figcaption>Sample sections.yml</figcaption>
~~~yaml
siteindex:
  name: Index
stat:
  name: Statistics
elem:
  name: Elements
~~~
</figure>

Breadcrumbs aren't typically necessary for sites, such as blogs, with a very
"flat" structure, where most pages are only one level down from the home
page.  Some sites have relatively large "flat" sections alongside
hierarchically organized content. It would be nice to show the breadcrumb
only for the hierarchical portions of the site. For the Cogitorium, I've
opted to toggle inclusion of the breadcrumb using a flag that can be set per
section using
[defaults](https://jekyllrb.com/docs/configuration/front-matter-defaults/)
or in the [front matter](https://jekyllrb.com/docs/front-matter/)
of individual pages. But, others might be interested in a programmatic
option. The following includes the breadcrumb only if the given page is at
least the third level down from the root:

<figure markdown="1"><figcaption>Conditional breadcrumb</figcaption>
{% raw %}
~~~liquid
{% assign crumbs = page.url | remove: '/index.html' | split: '/' %}
{% if crumbs.size > 2 %}
  {% comment %} Breadcrumb code here {% endcomment %}
{% endif}
~~~
{% endraw %}
</figure>

Note that sites that style blog post URLs with the year and month as
directories will automatically trigger the breadcrumb for all posts using
this method, and the links for the year and month crumbs will likely lead
only to ugly directory listings (if directory listing is enabled on the web
server). A slightly more nuanced approach to the conditional inclusion of
the breadcrumb would exclude posts explicitly:

<figure class="wide" markdown="1"><figcaption>Conditional breadcrumb, excluding posts</figcaption>
{% raw %}
~~~liquid
{% assign crumbs = page.url | remove: '/index.html' | split: '/' %}
{% if crumbs.size > 2 %}{% unless page.collection == "posts" %}
  {% comment %} Breadcrumb code here {% endcomment %}
{% endunless %}{% endif}
~~~
{% endraw %}
</figure>
