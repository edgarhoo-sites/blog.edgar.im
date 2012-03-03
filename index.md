---
layout: default
title: Edgar's Blog
---


{% for post in site.categories.article limit:1 %}
<article class="post">
    <header class="entry-header">
        <div class="entry-meta">
            <time class="entry-date">{{ post.date | date: "%B %d, %Y" }}</time>
        </div>
        <h1 class="entry-title"><a href="{{ post.url }}">{{ post.title }}</a></h1>
    </header>
    <div class="entry-content">
        {{ post.content }}
    </div>
    <footer class="entry-meta">
        Bookmark the <a href="http://blog.edgar.im{{ post.url }}" title="Permalink to {{ post.title }}" rel="bookmark">permalink</a>.		
    </footer>
</article>
{% endfor %}

{% if site.categories.article.size > 1 %}
    {% for rpost in site.categories.article limit:5 offset:1 %}
<article class="post recent-post">
    <div class="entry-header">
        <div class="entry-meta">
            <time class="entry-date">{{ rpost.date | date: "%B %d, %Y" }}</time>
        </div>
        <h1 class="entry-title"><a href="{{ rpost.url }}">{{ rpost.title }}</a></h1>
    </div>
</article>
    {% endfor %}
{% endif %}
