---
layout: page
title: 关于我, About
description: About
keywords: About
comments: true
menu: 关于
permalink: /about/
---

一个默默无闻的 Java Brick-Mover

## 联系方式

{% for website in site.data.social %}
* {{ website.sitename }}：[@{{ website.name }}]({{ website.url }})
{% endfor %}

## 技能列表

{% for category in site.data.skills %}
### {{ category.name }}
<div class="btn-inline">
{% for keyword in category.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
