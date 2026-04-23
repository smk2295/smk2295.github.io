---
layout: page
permalink: /publications/
title: publications
description:
nav: true
published: true
nav_order: 2
---

<!-- _pages/publications.md -->

<!-- Bibsearch Feature -->

{% include bib_search.liquid %}

<div class="publications">

{% bibliography --query @*[hidden!=true]* %}

</div>
