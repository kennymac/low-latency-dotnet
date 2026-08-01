---
layout: default
title: "Low-Latency .NET Tech Notes"
description: "Technical spikes and architecture notes on low-latency .NET, C#, Native AOT, Continuous Delivery and zero-defect systems."
---

# Low-Latency .NET Tech Notes

## Posts

{% assign technotes = site.pages | where_exp: "item", "item.path contains 'posts/'" | sort: "date" | reverse %}
{% for note in technotes %}
* **{{ note.date | date: "%Y-%m-%d" }}** — [{{ note.title }}]({{ note.url | relative_url }})  
  *{{ note.description }}*
{% endfor %}

---

### About

Written by **Kenneth McCormack** — distributed systems architect and principal engineer, specializing in .NET, Continuous Delivery, and zero-defect systems.
