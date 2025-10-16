---
layout: default
title: "전체 카테고리"
---

<style>
  .category-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 1.5rem;
    margin-top: 2rem;
  }

  .category-card {
    display: flex;
    flex-direction: column;
    border-radius: 12px;
    overflow: hidden;
    box-shadow: 0 4px 12px rgba(0,0,0,0.1);
    transition: transform 0.2s ease, box-shadow 0.2s ease;
    background-color: #fff;
    text-decoration: none;
    color: inherit;
  }

  .category-card:hover {
    transform: translateY(-6px);
    box-shadow: 0 8px 18px rgba(0,0,0,0.15);
  }

  .category-card img {
    width: 100%;
    height: 160px;
    object-fit: cover;
  }

  .category-card-content {
    padding: 1rem;
  }

  .category-card h2 {
    margin: 0 0 0.5rem 0;
    font-size: 1.3rem;
  }

  .category-card p {
    color: #555;
    font-size: 0.9rem;
    margin: 0;
  }
</style>

<h1>{{ page.title }}</h1>

<div class="category-grid">
  <!-- {% for category in site.categories %}
    {% assign cat_name = category[0] %}
    {% assign cat_posts = category[1] %}
  {% endfor %} -->
{{ site | inspect }}
</div>
