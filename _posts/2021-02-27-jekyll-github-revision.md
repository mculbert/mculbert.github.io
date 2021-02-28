---
title: Marking Revisions in Jekyll with GitHub
description: >
  A simple Javascript solution for displaying revision information for Jekyll
  posts using the GitHub API.
category: technology
tags: jekyll github javascript
---

[Jekyll](https://jekyllrb.com/) with
[GitHub Pages](https://pages.github.com/) provides a great setup for simple
static websites, like the Cogitorium. The clean simplicity of
[Markdown](https://en.wikipedia.org/wiki/Markdown) content combined with a
few simple template files stored in a [Git](https://git-scm.com/) repository
appeals to me.

Jekyll infers publication dates from posts based on their file names.  For
example, the file name for this post is
`2021-02-27-jekyll-github-revision.md` for February 27, 2021.  But, what
happens when a post is updated?  It would be nice if Jekyll could
automatically determine the revision date, either from the file's
modification time or from the Git commit log.  This is probably possible
with a custom plugin, but GitHub Pages only works with a specific, fixed set
of Jekyll plugins, a reasonable limitation due to security and server
resource considerations for a free service like this.  What to do?

Happily, the [GitHub API](https://docs.github.com/en/rest) provides
convenient machine-readable access to metadata for commits in Git
repositories, so a simple solution is possible via Javascript.  Although I
generally wanted to eliminate as much Javascript as possible from the new
design of the Cogitorium to keep things clean, minimal, and zippy, I figured
this simple call at the bottom of the page would be a fine compromise.

This solution relies on an empty `div` where the revision information should
go (in the footer, in this case):

<figure markdown="1"><figcaption>Destination</figcaption>
~~~ html
<footer><div class="revision"></div></footer>
~~~
</figure>

I chose to identify the target `div` via the `revision` class, which made
styling via CSS easy, but the target element could probably just as easily
be identified by `id`.

Then, the Javascript to fetch, parse, and display the revision information
for the given post:

<figure class="wide" markdown="1"><figcaption>Fetch revision from GitHub</figcaption>
{% raw %}
~~~ javascript
fetch("https://api.github.com/repos/{{ site.github.repository_nwo }}/commits?path={{ page.path }}&page=1&per_page=2")
  .then((response) => { return response.json(); })
  .then((json) => {
    if (json.length > 1) {
      var d = new Date(json[0].commit.author.date);
      document.querySelector("div.revision").innerHTML = "Revised " +
        d.toLocaleString() + ": " +
        json[0].commit.message + " (<a href='{{ site.github.repository_url }}/commits/main/{{ page.path }}'>history</a>)";
    } });
~~~
{% endraw %}
</figure>

The first line calls the GitHub API to
[list the commits](https://docs.github.com/en/rest/reference/repos#list-commits)
for the given post, which is specified by the `page.path`
[Jekyll variable](https://jekyllrb.com/docs/variables/), seemingly made
precisely for this purpose.  The call asks for the first page (`page=1`) of
the results paginated in twos (`per_page=2`), effectively limiting the commits
returned to the two most recent.  The next line parses the
[JSON](https://en.wikipedia.org/wiki/JSON) response into a Javascript data
structure.  I decided to display the commit information only if a revision
has been made (after the original post), so we test to see if there is more
than one commit before outputting any results.  The rest formats the
revision date and commit message, along with a link to the commit log for
the file on GitHub.  Finally, the HTML-formatted revision text is stuffed
into the content of the `div.revision` target element.

Piece of cake!
