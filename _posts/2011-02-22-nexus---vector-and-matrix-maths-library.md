---
title:   "Nexus - vector and matrix maths library for .NET / Silverlight, suitable for 3D development"
date:    2011-02-22 09:51:00 UTC
---

[Nexus](https://github.com/tgjones/nexus) is a vector and maths library I have been using for my own projects for a while now. 

Unlike WPF's 3D structures in the `System.Windows.Media.Media3D` namespace, such as `Point3D`, the structures in Nexus use fields, rather than properties, so they can be used directly in the context of vertex shaders and pixel shaders, which require predicable structure layouts. 

I followed WPF's example of having separate structures for points (positions) and vectors (directions, i.e. normals). I feel this makes for cleaner APIs.

Nexus has structures for:

* 2D, 3D and 4D vectors and points
* 3D lines and rays
* Quaternion class
* 3x3 and 4x4 matrices
* Colors
* Bounding boxes
* Transforms (scale, translation, rotation)

Nexus also includes a `Camera` class, with `PerspectiveCamera` and `OrthographicCamera` derived classes. `PerspectiveCamera` includes a `CreateFromBounds()` method that might be interesting to others, which allows the camera to be positioned based on a bounding box of an object or scene.

Nexus has no dependencies apart from the .NET framework, is available for .NET 4.0 and Silverlight 4, and is open source. You can get the binaries from two locations: 

* [GitHub](https://github.com/tgjones/nexus/downloads)
* [NuGet](http://www.nuget.org/Packages/Packages/Details/Nexus-1-0-0-0)

I wrote Nexus because some of my graphics projects don't use XNA, and I wanted a standard (and lightweight) maths library I could fall back on in those situations. Obviously if your game or project targets XNA or another framework that comes with its own maths library, then you wouldn't need something like Nexus.