---
layout: page
title: "Notes"
permalink: /notes/
---

{% assign grouped = site.notes | group_by: "folder" | sort: "name" %}

<ul class="notes-list">
{% for group in grouped %}
  {% if group.name != "" %}
  <li class="notes-folder">
    <span class="notes-folder-name">{{ group.name }}/</span>
    <ul class="notes-subitems">
      {% for note in group.items %}
      <li><a href="{{ note.url }}">{{ note.title }}</a></li>
      {% endfor %}
    </ul>
  </li>
  {% else %}
    {% for note in group.items %}
  <li><a href="{{ note.url }}">{{ note.title }}</a></li>
    {% endfor %}
  {% endif %}
{% endfor %}
</ul>

