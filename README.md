# Coding Nights
Coding Nights is powered by a modified version of the [Centrarium Theme](https://github.com/bencentra/centrarium).

## Changes

Main changes regard the website `style`. CSS files have been modified to:

- make the home page's header image a little smaller so that the underlying text results more visible;
- change the website color combination (see `_variable.scss`);
- customize the footer content to display new variables added to `_config.yml`;
- style block quotes (see `_typography.scss`) - missing in the base theme;
- style caption for images (see `_layout.scss`) - missing in the base theme;
- change the syntax highlighting to use `Rouge` only (see `_code.scss` and `_highlight.scss`).

## Fixes
The file `header.html` has been changed in order not to display categories and tags coming from the
generation of `jekyll-archives`. The change excludes from the navigation menu any page with the `archive` layout.

```
{% for page in site.pages %}
  {% unless page.layout contains 'archive' %}
    {% if page.title and page.main_nav != false %}
      <li class="nav-link"><a href="{{ page.url | prepend: site.baseurl }}">{{ page.title }}</a></li>
    {% endif %}
  {% endunless %}
{% endfor %}
```

## Syntax Highlighting
[Rouge](http://rouge.jneen.net/), the default syntax highlighter in Jekyll 3, doesn't automatically style your code. 
To achieve the results in Coding Nights, [Alex Peattie's blog post](https://alexpeattie.com/blog/better-syntax-highlighting-with-rouge) 
and [repository](https://github.com/alexpeattie/alexpeattie.com) were helpful and inspiring.
