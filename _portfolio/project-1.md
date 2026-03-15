---
title: "pitzDailySteady BFS (Backward-Facing Step)"
permalink: /portfolio/pitzdailysteady-bfs/
excerpt: "Reattachment-length extraction on `pitzDailySteady` with a reproducible post-processing pipeline and turbulence-model sensitivity analysis."
header:
  overlay_image: /assets/images/project-1-cover.svg
  overlay_filter: 0.35
  teaser: /assets/images/project-1-cover.svg
  actions:
    - label: "GitHub Repo"
      url: https://github.com/1001wngud/CFD-Study-BFS
    - label: "Progress Logs"
      url: "#progress-logs"
toc: true
toc_sticky: true
classes: wide
---

## Summary

This project turns the OpenFOAM tutorial `pitzDailySteady` into a **QoI-driven BFS benchmark study**.
Instead of only running a tutorial case, the workflow defines the main quantity of interest as the nondimensional reattachment length `xr/H`, fixes a reproducible post-processing rule, and compares turbulence-model sensitivity between `kEpsilon` and `kOmegaSST`.

The reattachment position is extracted from near-wall `Ux(x)` data and cross-checked against the formal wall-shear definition `tau_w = 0`. This makes the project useful as both a CFD study and a portfolio example of **problem definition → extraction rule → validation → comparison**.

## Research question

How can the reattachment length `xr/H` in a backward-facing step flow be defined and extracted **reproducibly**, and how sensitive is that result to the choice of turbulence model?

## Numerical setup

- **Case:** OpenFOAM `pitzDailySteady`
- **Flow type:** steady incompressible RANS
- **Algorithm:** SIMPLE
- **Fluid:** Newtonian, `nu = 1e-05 m^2/s`
- **Inlet velocity:** `10 m/s`
- **Reference length:** step height `H = 0.0254 m`
- **Reynolds number:** `Re_H = 25400`
- **Models compared:** `kEpsilon`, `kOmegaSST`
- **Main QoI:** reattachment length `xr/H`

## Key result

- `kOmegaSST` predicts a reattachment length about **16.7–17.2% longer** than `kEpsilon`.
- The near-wall `Ux = 0` estimate is **consistent with** the wall-shear-based `tau_w = 0` definition for the baseline case.
- Sampling height changes the extracted zero-crossing systematically, so the project explicitly checks **line-position sensitivity**.
- The final workflow is organized as a reusable extraction pipeline instead of a one-time plot inspection.

## What this project shows about my CFD workflow

- defining a meaningful **quantity of interest** from a canonical flow
- building a **reproducible post-processing rule**
- checking agreement between **physical definition** and **numerical proxy**
- comparing model sensitivity with a consistent measurement pipeline

## GitHub repository

[Open the repository](https://github.com/1001wngud/CFD-Study-BFS){: .btn .btn--primary}

## Progress logs

{:#progress-logs}
1. [01. Track A README — benchmark context and QoI definition]({{ "/bfs/01-tracka-readme/" | relative_url }})
2. [02. PROJECT_FLOW — reproducibility and workflow structure]({{ "/bfs/02-project-flow/" | relative_url }})
3. [03. report_Day12 — quantitative report and extracted tables]({{ "/bfs/03-day12-report/" | relative_url }})
4. [04. LAB_CONTACT_1PAGE — one-page summary for lab contact]({{ "/bfs/04-lab-contact-summary/" | relative_url }})

> This page is already structured to match the four detailed BFS markdown files you mentioned. You can replace each page below with the actual contents from your own notes.
