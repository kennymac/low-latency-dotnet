---
layout: default
title: "Low-Latency .NET Tech Notes"
description: "Technical spikes and architecture notes on low-latency .NET, C#, Native AOT, Continuous Delivery and zero-defect systems."
---

<div class="row align-items-center mb-5 profile-header">
  <div class="col-12">
    <h1 class="display-6 fw-bold mb-2">Kenneth McCormack</h1>
    <p class="lead text-muted mb-3">Distributed Systems Architect & Principal Engineer</p>
    <p class="bio-text">
      Specializing in low-latency .NET, C#, Native AOT compilation, Continuous Delivery, and zero-defect systems.
    </p>
    <div class="social-links mt-3">
      <a href="https://github.com/kennymac" target="_blank" class="btn btn-outline-secondary btn-sm me-2">
        <i class="fab fa-github me-1"></i> GitHub
      </a>
    </div>
  </div>
</div>

<hr class="my-5 border-color">

<section id="posts" class="posts-section">
  <h3 class="fw-bold mb-4">Posts & Technotes</h3>

  <div class="posts-list">
    {% assign technotes = site.pages | where_exp: "item", "item.path contains 'posts/'" | sort: "date" | reverse %}
    {% for note in technotes %}
    <article class="post-card">
      <h4 class="post-card-title">
        <a href="{{ note.url | relative_url }}">{{ note.title }}</a>
      </h4>
      <p class="post-card-description">{{ note.description }}</p>
      <div class="post-card-meta">
        <i class="far fa-calendar me-1"></i> {{ note.date | date: "%B %d, %Y" }}
      </div>
    </article>
    {% endfor %}
  </div>
</section>
