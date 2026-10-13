# WebGPU Forward+ and Clustered Deferred Shading

University of Pennsylvania, CIS 565: GPU Programming and Architecture, Project 4

- Sizhe Liu
- Tested on: Windows 11 Pro, Intel Core i9-12900H @ 2.50 GHz, 32 GB RAM, NVIDIA RTX 3090 24 GB (personal computer)
- Chrome 155.0.8059.40

## Demo

[Live demo](https://jamesliu12.github.io/Project4-WebGPU-Forward-Plus-and-Clustered-Deferred/)

![Sponza with 2000 point lights](img/gallery.png)

Sponza with 2000 moving point lights.

## Overview

I implemented Forward+ and Clustered Deferred rendering in WebGPU. Both assign lights to a 16×9×24 grid of view-space clusters, so each shaded pixel only evaluates nearby lights. Forward+ shades geometry directly, while Deferred stores surface attributes in a G-buffer and shades visible pixels in a fullscreen pass.

The extra credit features are a packed G-buffer with compute lighting and a depth prepass with depth-based cluster culling for Forward+.

## Performance Analysis

Timings measure GPU render passes in milliseconds, reported as medians across three runs; error bars show the range. Comparisons use the same Sponza scene and light state, with a fixed camera, cluster grid, and light radius of 2.

### Forward+ vs. Clustered Deferred

*Varying lights: 500, 1000, 2000, 5000. Fixed: 2560×1440, atrium view.*

![Light count scaling at 2560×1440](img/light-scaling.png)

| Lights | Forward+ (ms) | Deferred (ms) | Deferred speedup |
| --- | --- | --- | --- |
| 500 | 5.505 | 1.704 | 3.23× |
| 1000 | 9.601 | 3.932 | 2.44× |
| 2000 | 19.071 | 6.193 | 3.08× |
| 5000 | 47.448 | 13.926 | 3.41× |

At **2560×1440**, Deferred is **2.44–3.41× faster** than basic Forward+ across 500–5000 lights. With 5000 lights, rendering time drops from **47.45 to 13.93 ms**. Shading visible pixels once avoids much of Forward+'s repeated lighting work on overlapping geometry.

Light assignment takes O(clusters × lights), and shading depends on the lights in each pixel's cluster. Both costs grow with light count. Deferred adds G-buffer storage, while Forward+ avoids that storage; a depth prepass can reduce its overdraw cost.

*At 5000 lights, some visible clusters hit the 1024-light limit and truncate their lists. All variants share this cap; 500–2000 lights stay below it.*

### G-buffer Optimization

I pack normals and albedo into one `rg32uint` texture and reconstruct position from depth. This reduces color G-buffer storage from **20 to 8 bytes per pixel**, saving **60%**: **70.31 → 28.13 MiB** at 1440p, excluding depth.

*Varying resolution: 1920×1080, 2560×1440, 3840×2160. Fixed: 5000 lights, atrium view.*

![G-buffer and compute lighting comparison at 5000 lights](img/resolution-scaling.png)

| Resolution | Base G-buffer + fragment | Packed + fragment | Packed + compute |
| --- | --- | --- | --- |
| 1920×1080 | 8.684 ms | 8.684 ms | 9.142 ms |
| 2560×1440 | 13.926 ms | 14.156 ms | 14.811 ms |
| 3840×2160 | 28.410 ms | 28.705 ms | 30.999 ms |

Packing does not improve total rendering time here: it matches the baseline at 1080p and takes **1.6% / 1.0% longer** at 1440p / 4K. Extra reconstruction and decoding can offset the savings from smaller attachments.

I also added an 8×8 compute lighting pass that writes directly to the canvas. It is **4.6–8.0% slower** than packed fragment lighting in this sweep. Moving lighting to compute still leaves the same per-pixel light evaluations.

### Depth Prepass for Forward+

The depth prepass lets the lighting pass reject hidden surfaces before shading. Depth-based culling then marks visible clusters and skips light assignment for inactive ones.

*Compare basic Forward+, prepass only, and prepass + culling at 2560×1440 with 500 and 5000 lights. View A: atrium; view B: side corridor.*

![Depth prepass and cluster culling comparison](img/prepass-culling.png)

| View / lights | Forward+ | + Prepass | + Culling | Prepass time reduction |
| --- | --- | --- | --- | --- |
| A / 500 | 5.505 ms | 1.835 ms | 2.818 ms | 66.7% |
| A / 5000 | 47.448 ms | 17.695 ms | 18.842 ms | 62.7% |
| B / 500 | 2.621 ms | 2.294 ms | 3.178 ms | 12.5% |
| B / 5000 | 31.064 ms | 10.977 ms | 12.059 ms | 64.7% |

The prepass reduces rendering time by **12.5–66.7%**, with the smallest gain in the side corridor at 500 lights and the largest in the atrium at 500 lights. In the atrium with 5000 lights, time drops from **47.45 to 17.69 ms**. Avoiding repeated lighting outweighs the extra geometry pass here; scenes with less overdraw may benefit less.

Adding culling is **6.5–53.6% slower** than the prepass alone, even though only about **8% of clusters** are active. The marking pass costs more than the clustering work it saves. My implementation skips inactive clusters with early returns but keeps the full dispatch.

### Where the Time Goes

![GPU pass breakdown at 1440p and 5000 lights](img/pass-breakdown.png)

Mean GPU pass times at 2560×1440 with 5000 lights in the atrium. Lighting dominates the workload, which explains why reducing overdraw helps more than reducing cluster assignment work.

## Run Locally

```sh
npm install
npm run dev
```

## Credits

- [Vite](https://vitejs.dev/)
- [loaders.gl](https://loaders.gl/)
- [dat.GUI](https://github.com/dataarts/dat.gui)
- [stats.js](https://github.com/mrdoob/stats.js)
- [wgpu-matrix](https://github.com/greggman/wgpu-matrix)
