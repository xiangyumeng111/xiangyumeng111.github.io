---
layout: page
title: projects
permalink: /projects/
description: Research and engineering projects.
nav: true
nav_order: 2
display_categories: []
horizontal: true
---

<!-- pages/projects.md -->

<style>
.projects .project-horizontal-card.has-project-image {
  display: grid;
  grid-template-columns: minmax(260px, 42%) 1fr;
  gap: 0;
  align-items: center;
  overflow: hidden;
}
.projects .project-horizontal-media {
  padding: 1.1rem 0 1.1rem 1.1rem;
}
.projects .project-horizontal-media figure {
  margin: 0;
}
.projects .project-horizontal-img {
  display: block;
  width: 100%;
  height: auto;
  border-radius: 8px;
  border: 1px solid var(--global-divider-color);
  background: var(--global-bg-color);
}
.projects .project-horizontal-body {
  padding: 1.25rem 1.45rem;
}
@media (max-width: 760px) {
  .projects .project-horizontal-card.has-project-image {
    display: block;
  }
  .projects .project-horizontal-media {
    padding: 1rem 1rem 0;
  }
  .projects .project-horizontal-body {
    padding: 1rem 1.15rem 1.15rem;
  }
}
</style>

<div class="projects">
{% if site.enable_project_categories and page.display_categories %}
  <!-- Display categorized projects -->
  {% for category in page.display_categories %}
  <a id="{{ category }}" href=".#{{ category }}">
    <h2 class="category">{{ category }}</h2>
  </a>
  {% assign categorized_projects = site.projects | where: "category", category %}
  {% assign sorted_projects = categorized_projects | sort: "importance" %}
  <!-- Generate cards for each project -->
  {% if page.horizontal %}
  <div class="container">
    <div class="row row-cols-1">
    {% for project in sorted_projects %}
      {% include projects_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for project in sorted_projects %}
      {% include projects.liquid %}
    {% endfor %}
  </div>
  {% endif %}
  {% endfor %}

{% else %}

<!-- Display projects without categories -->

{% assign sorted_projects = site.projects | sort: "importance" %}

  <!-- Generate cards for each project -->

{% if page.horizontal %}

  <div class="container">
    <div class="row row-cols-1">
    {% for project in sorted_projects %}
      {% include projects_horizontal.liquid %}
    {% endfor %}
    </div>
  </div>
  {% else %}
  <div class="row row-cols-1 row-cols-md-3">
    {% for project in sorted_projects %}
      {% include projects.liquid %}
    {% endfor %}
  </div>
  {% endif %}
{% endif %}
</div>
