---
layout: home
title: Notes
---

Field notes from side projects and experiments. Mostly software, agents,
and infrastructure. Honest about what worked and what didn't.

## Posts

<ul class="post-list">
{% for post in site.posts %}
  <li>
    <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span>
    <h3>
      <a class="post-link" href="{{ post.url | relative_url }}">{{ post.title | escape }}</a>
    </h3>
    {% if post.subtitle %}<p class="post-subtitle"><em>{{ post.subtitle }}</em></p>{% endif %}
  </li>
{% endfor %}
</ul>

---

[About]({{ "/about" | relative_url }}) · [RSS]({{ "/feed.xml" | relative_url }})
