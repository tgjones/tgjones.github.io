---
title:   "Osiris SceneGraph Development Part 1 - Introduction"
date:    2008-07-29 21:00:00 UTC
---

This series will describe the development of a scene graph toolkit. This is a non-work project so I can't promise regular updates, but I will do my best.

## Motivation

I wondered about calling this YASG (Yet Another Scene Graph). <a href="http://www.openscenegraph.org/">There's</a> <a href="http://www.opensg.org/">quite</a> <a href="http://developer.nvidia.com/object/nvsg_home.html">a</a> <a href="http://www.ogre3d.org/">few</a> of them out there. However, nothing quite suited.

My primary goal in writing a scene graph is to use it as the basis of an XNA game. However, I've got some other projects on the go. One is a <a href="http://www.roastedamoeba.com/blog/archive/2008/05/15/xna--hybrid-ray-tracer">ray tracer</a>, and the other is a software renderer. The software renderer will be used in an ASP.NET control that I am developing for my company.

I realised that I was writing very similar code for each of these three projects: specifically, code to manage the scene data. All three required code for some sort of tree involving transforms; all three required code for camera transformations; all three required code for storing triangular meshes.

It seemed to me that it would be highly desirable to completely separate the scene data from the rendering. If this was possible, I could write one scene graph system, and three renderers (ray tracer, hardware rasterizer, software rasterizer).

If at this point you're thinking "uh oh - I see where this is going - it's going to be a nightmare to abstract away the differences between those renderers", then you'd be completely right. And if this was a commercial project, it would almost certainly be dead in the water, in terms of reward for time spent. But this is a hobby project, and I love abstraction.

## Problems

Immediately I began to hit problems. Possibly the most basic requirement of any graphics library is a good set of maths classes - i.e. vector, matrix, etc. Since the primary usage of the scene graph is going to be for an XNA game, it would make sense to use the maths classes available in XNA (Vector3, Matrix, etc.) right? Sadly not. I am exposing the other two renderers (software rasterizer, ray tracer) through an ASP.NET web control. I don't want to have Microsoft.Xna.Framework as a dependency for an ASP.NET control. The same goes for the maths classes in WPF (Vector3D, Point3D, etc.).

## Core classes

So I decided to write my own maths classes. Having now done this, I can say it's not fun, when it feels so much like re-inventing the wheel. But I wanted a scene graph which could be used by multiple renderers. So I don't have much choice. I used both WPF (the Media3D namespace) and XNA as a model for my maths classes. I like how WPF separates out Vector3D and Point3D. I know you can accomplish everything you need to with just a Vector3D, but it seems more rigorous to provide separate types. I have implemented the following types:

* Matrix3 (used for some physics calculations)
* Matrix4
* Plane
* Point (for texture coordinates)
* Point3D
* Ray3D
* Vector3D

## Scene graph

Whew! I haven't even started describing the design of the scene graph but I'm going to finish soon. In summary: in this series of blog posts I will be describing the development of a scene graph, starting from scratch. I will try to describe the major problems I face along the way. Later on I will talk about how I've implemented / am implementing each of the three renderers (ray tracer, software rasterizer, hardware rasterizer).

I did quite a lot of research of existing scene graphs (including OpenSceneGraph and Nvidia Scene Graph), and tried to distil what I liked from them, together with my own ideas. I also had the restriction of needing the scene graph to work with a ray tracer, which is a limitation not faced by most existing scene graphs (which are optimised for a particular renderer, such as OpenGL). At the same time, the scene graph needed to cope with a hardware renderer, which (for example) requires vertices to be stored in a vertex buffer in graphics memory. I will describe the solutions I came up with in Part 2.