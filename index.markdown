---
layout: post
title: Jean-Michel Lacroix
nofoot: true
---

Full-stack developer with a strong interest in products, user experience and technologies (web, mobile, cloud, languages, systems).

<a href="mailto:&nbsp;j&nbsp;e&nbsp;&#64;&nbsp;n&nbsp;&#45;m&nbsp;i&nbsp;&#46;&nbsp;c&nbsp;h">Email</a>,
[Twitter](http://twitter.com/jmlacroix),
[Github](http://github.com/jmlacroix),
[Linked In](http://linkedin.com/in/jmlacroix),
[Speaker Deck](http://speakerdeck.com/u/jmlacroix)

Articles
--------

<ul>
{% for post in site.posts %}
  {% unless post.draft %}
  <li><a href="{{ post.url }}">{{ post.title }}</a></li>
  {% endunless %}
{% endfor %}
</ul>

Talks
-----

 - <a href="/archives/ipfh.html">Introduction à la programmation fonctionnelle avec Haskell</a>
