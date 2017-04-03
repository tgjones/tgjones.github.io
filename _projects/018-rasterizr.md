---
title: "Rasterizr"
permalink: /projects/rasterizr
excerpt: "A software rasterizer written from scratch in C#, capable of executing real HLSL shaders on the CPU."
year: 2013
teaser: /assets/projects/rasterizr-environment-mapping.png
gallery:
  - /assets/projects/rasterizr-environment-mapping.png
  - /assets/projects/rasterizr-debugging.png
---

A software rasterizer written from scratch in C#. Since software rasterizers have limited use in real products, I have emphasised code clarity over performance, with a view to writing some tutorials about rolling your own rasterizer. Again, I don't expect anybody will really use their own rasterizer, but personally I find it very useful to know what's going on under the hood.

The API is closely modelled on Direct3D 10 and 11, and is split into several pipeline stages: input assembler, vertex shader, geometry shader, rasteriser, pixel shader and output merger.

Rasterizr uses [SlimShader](/projects/slimshader) to parse and execute HLSL shaders entirely on the CPU, in managed code.

* [Source Code](https://github.com/tgjones/rasterizr) (.NET 4.0)