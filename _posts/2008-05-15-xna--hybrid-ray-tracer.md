---
title:   "XNA 3.0 Hybrid Ray Tracer"
date:    2008-05-15 23:49:00 UTC
---

* [XNA Hybrid Tracer Source Code](https://github.com/tgjones/rasteracer) (updated to XNA 4.0)

Last week while going through my personal software graveyard, I "dug up" a ray tracer I started a few years ago. I had written it using C#, and it was rendered using GDI+. I decided to bring it up to date - and you can't get newer than the XNA 3.0 CTP, released last week. At this stage there's hardly any new features related to Windows development, apart from audio, but I like being able to use Visual Studio 2008 and LINQ.

Anyway, one thing that annoyed me about my previous implementation was how difficult it was to move the camera around. So, I decided to write a hybrid ray tracer:

* When the camera is static, ray tracing is used.
* When the camera is moving, XNA hardware rasterization is used.

## Features

* Automatically switches from ray tracing to hardware rasterization when camera is moving
* Loads scenes from an XML file using the XNA Content Pipeline
* Supports sphere and plane primitives, with other primitives coming soon (including triangles, and hence meshes)
* Point lights, with other light types coming soon

The current version is fairly basic, both in terms of ray tracing and rasterization. The two render methods are also not very similar - about the only thing they are the same at is object position. However, I know it's usually better for me to upload *something*, however basic, just to get the ball rolling. So here it is. Feel free to use the code in any way you like. Hopefully you'll find it logically structured.

The camera has two modes:

* Pan: This took me a while to get right. I eventually realised that software like 3D Studio Max converts mouse coordinates from screen space to world space, and then pans in world space. So I did the same.
* Arc ball: I'm not happy with my implementation of this. It sort of works though - I set it to rotate around a point 5 world units in front of the camera.
* In either mode, you can use the mouse scroll wheel to zoom (although this doesn't work very well yet).

My todo list:

* Ray tracer
  * Refraction
  * Photon mapping
  * Textures
  * kd-tree for spatial sorting
* XNA rasterization
  * Specular lighting
  * Environment mapping
  * Planar reflections and refractions
* Both renderers
  * Fresnel effect
  * Textures

## Screenshots

![](/assets/posts/raytrace1.png)

![](/assets/posts/xna1.png)

![](/assets/posts/raytrace2.png)

![](/assets/posts/xna2.png)