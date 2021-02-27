---
layout: postx
title: "Orion Nebula"
tags: ["orion nebula", "m42"]
---
Got an image last evening of the [Orion Nebula](https://en.wikipedia.org/wiki/Orion_Nebula)
that I'm quite satisfied with.

{% include figure.html url="assets/img/M42 - Orion Nebula - 2021-02-26.jpg" description="Orion Nebula - M42" %}

# Technical details

The image is taken with a 200-500mm ƒ/5.6mm lens on a regular tripod. Here's
the setup:

| Focal length | 200mm |
| Aperture | ƒ/5.6 |
| ISO | 8000 |
| Exposure time for light frames | 350 * 1s |
| Dark frames | 50 |
| Flat frames | 50 |
| Bias frames | 50 |

The raw photos were stacked using [DSS](http://deepskystacker.free.fr/english/index.html),
and then post-processed in [Siril](https://www.siril.org/). First used
Photometric Color Calibration to get color correction, then auto stretching with
manual tweaking. After that some background extraction, and finally some
Contrast Limited Adaptive Histogram Equalization.
