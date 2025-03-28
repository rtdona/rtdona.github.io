---
icon: fas fa-bookmark
order: 5
title: "My Bookmarks"
---

> BookMarks


## 📌 북마크 목록

{% for bookmark in site.bookmarks %}
- 🔖 [{{ bookmark.title }}]({{ bookmark.url }}) ({{ bookmark.date | date: "%Y-%m-%d" }})
  {% endfor %}
