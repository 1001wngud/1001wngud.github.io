---
title: "CFD Study of IGE (In-Ground Effect)"
permalink: /portfolio/ige-in-ground-effect/
excerpt: "OpenFOAM v13 `propellerDisk` hover-IGE study comparing performance ratios and outwash-peak movement across height ratios `h/R`."
header:
  overlay_image: /assets/images/project-2-cover.svg
  overlay_filter: 0.35
  teaser: /assets/images/project-2-cover.svg
  actions:
    - label: "GitHub Repo"
      url: https://github.com/1001wngud/CFD-Study-IGE
    - label: "Progress Logs"
      url: "#progress-logs"
toc: true
toc_sticky: true
classes: wide
---

## Summary

This project studies **hover in-ground effect (IGE)** using OpenFOAM Foundation v13 with the `propellerDisk` fvModel and a `createZones`-based mesh-zone workflow.
The focus is not matching one commercial propeller exactly, but organizing a consistent CFD workflow to compare how normalized performance and near-ground outwash behavior change as the height ratio `h/R` decreases.

The result is a parameter-comparison study centered on **trend interpretation, metric design, and sensitivity-aware analysis**.

## Research question

- How do normalized performance metrics `T/Tref` and `P/Pref` change as `h/R` decreases?
- How does the near-ground outwash peak location `rmax/R` move with decreasing height?
- How sensitive are the low-altitude results to the assumed low-`J` curve?

## Numerical setup

- **Framework:** OpenFOAM Foundation v13
- **Rotor model:** `propellerDisk` fvModel
- **Zone workflow:** `createZones`-based mesh-zone setup
- **Condition:** hover IGE
- **Parameter sweep:** multiple height ratios `h/R`
- **Main comparison metrics:** `T/Tref`, `P/Pref`, `rmax/R`, and `Vr,peak`

## Key result

- As `h/R` decreases, the near-ground outwash peak `rmax/R` moves **inward**.
- In the current setup, `T/Tref` and `P/Pref` show **saturation-like behavior** for `h/R <= 1`.
- The extracted `rmax/R` is identical at `h/R = 0.5` and `0.35`, suggesting a low-altitude plateau within the current post-processing workflow.
- That plateau interpretation remains **conditional**, because the low-altitude region is sensitive to the assumed low-`J` curve.

## What this project shows about my CFD workflow

- applying **propeller-disk / actuator-type modeling** in OpenFOAM
- organizing a **parameter sweep** around a physically meaningful control variable
- designing a comparison metric for **near-ground outwash**
- interpreting trends while keeping **model sensitivity** explicit

## GitHub repository

[Open the repository](https://github.com/1001wngud/CFD-Study-IGE){: .btn .btn--primary}

## Progress logs

{:#progress-logs}
1. [01. Overview, objective, and comparison metrics]({{ "/ige/01-overview/" | relative_url }})
2. [02. Propeller-disk setup, zones, and solver workflow]({{ "/ige/02-setup/" | relative_url }})
3. [03. Performance trends, outwash results, and sensitivity notes]({{ "/ige/03-results/" | relative_url }})

> Replace these example log pages with your own markdown records if you already organized the project in detail.
