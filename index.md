---
layout: splash
permalink: /
title: "JOO's CFD Study"
excerpt: "A clean, repo-first CFD portfolio for graduate lab applications."
header:
  overlay_image: /assets/images/hero-cfd.svg
  overlay_filter: 0.45
  actions:
    - label: "View Projects"
      url: /projects/

feature_row:
  - image_path: /assets/images/project-1-cover.svg
    alt: "pitzDailySteady BFS cover"
    title: "pitzDailySteady BFS (Backward-Facing Step)"
    excerpt: "QoI-driven BFS study in OpenFOAM: reproducible extraction of reattachment length `xr/H`, cross-check with `tau_w = 0`, and turbulence-model sensitivity between `kEpsilon` and `kOmegaSST`."
    url: /portfolio/pitzdailysteady-bfs/
    btn_label: "Open Project"
    btn_class: "btn--primary"

  - image_path: /assets/images/project-2-cover.svg
    alt: "CFD Study of IGE cover"
    title: "CFD Study of IGE (In-Ground Effect)"
    excerpt: "Hover IGE study using OpenFOAM v13 `propellerDisk`: compare normalized performance and near-ground outwash trends as the height ratio `h/R` decreases."
    url: /portfolio/ige-in-ground-effect/
    btn_label: "Open Project"
    btn_class: "btn--primary"

workflow_row:
  - title: "Repo-first pages"
    excerpt: "Each project page opens with a short technical summary and a **GitHub repository button** near the top."

  - title: "Detailed markdown logs"
    excerpt: "After the summary, readers can jump into **step-by-step progress logs** written in markdown."

  - title: "Built for lab contact"
    excerpt: "The structure is optimized for fast scanning: **problem → method → result → repository → logs**."
---

Welcome to **JOO's CFD Study**.

This website is a compact portfolio for graduate-lab contact. It currently highlights two CFD projects:

- a **Backward-Facing Step (BFS)** study based on `pitzDailySteady`
- a **hover In-Ground Effect (IGE)** study using `propellerDisk` in OpenFOAM v13

The homepage gives a short overview, each project page explains the technical work in a concise way, and the detailed workflow is organized in markdown progress logs.

{% include feature_row %}

## How to read this site

{% include feature_row id="workflow_row" type="center" %}
