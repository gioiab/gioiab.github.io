---
layout: post
title:  "Building something from scratch"
date:   2015-11-22 00:00:00
author: Gioia Ballin
categories: life blogging
tags:	blogging
cover:  "/assets/worker.jpg"
---


{% highlight ruby %}
def foo
  puts 'foo'
end
{% endhighlight %}


```html
<div class="post-header-container {% if page.cover %}has-cover{% endif %}" {% if page.cover %}style="background-image: url({{ page.cover | prepend: site.baseurl }});"{% endif %}>
  <div class="scrim {% if page.cover %}has-cover{% endif %}">
    <header class="post-header">
      <h1 class="title">{{ page.title }}</h1>
      <p class="info">by <strong>{{ page.author }}</strong></p>
    </header>
  </div>
</div>
```
