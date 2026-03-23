---
title: Listen & Subscribe
layout: page
permalink: /listen.html
---

## Subscribe to Open Invitation

Listen on your favorite podcast app:

{% include feature/subscribe-badges.html %}

## RSS Feed

Copy this URL into any podcast app to subscribe directly:

<div class="input-group mb-3" style="max-width:600px;">
  <input type="text" class="form-control" value="{{ site.url }}{{ site.baseurl }}/feed.xml" id="rss-url" readonly>
  <button class="btn btn-outline-secondary" onclick="document.getElementById('rss-url').select(); navigator.clipboard.writeText(document.getElementById('rss-url').value); this.textContent='Copied!'; setTimeout(()=>this.textContent='Copy',2000);">Copy</button>
</div>

## All Episodes

{% assign episodes = site.data[site.metadata] | where_exp: "item", "item.object_location != nil and item.object_location != ''" | sort: "date" | reverse %}
{% for item in episodes %}
### {{ item.title }}
{% if item.guest %}*with {{ item.guest }}*  {% endif %}
{{ item.date | date: "%B %d, %Y" }}{% if item.duration %} · {{ item.duration }}{% endif %}

{{ item.description | truncatewords: 30 }}

[Listen S{{ item.season_number}}E{{ item.episode_number }} →]({{ '/items/' | append: item.objectid | append: '.html' | relative_url }})

---
{% endfor %}
