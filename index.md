---
layout: default
title: Home
---

{% for post in site.posts %}
- [{{ post.title }}]({{ post.url }}) — {{ post.date | date: "%Y-%m-%d" }}
{% endfor %}

## 학습 기록
{% assign notes = site.til | sort: "date" | reverse %}
{% for note in notes %}
- [{{ note.title }}]({{ note.url }}) — {{ note.date | date: "%Y-%m-%d" }}
{% endfor %}
