---
title:   "Meshellator - Open source 3D asset import library for .NET / Silverlight"
date:    2011-02-22 22:48:00 UTC
---

[Meshellator](https://github.com/tgjones/meshellator) is an open source 3D asset import library, available for both .NET 4.0 and Silverlight 4.0. Meshellator is the underlying library for the [3DS, OBJ and NFF content importers for XNA](/blog/archive/2010/11/20/ds-obj-and-nff-content-importers-for-xna) which I have mentioned previously.

## What?

Supported file formats are quite limited at the moment. If anybody wants to contribute more, please [feel free](https://github.com/tgjones/meshellator)!

* 3DS
* OBJ
* NFF

As well as importing meshes from a file, Meshellator also allows you to create meshes from primitive shapes:

* Cube
* Cylinder
* Plane
* Sphere
* Teapot
* Torus

Meshellator's only dependency is on [Nexus](https://github.com/tgjones/nexus), a lightweight vector and matrix library I [blogged about recently](/blog/archive/2011/02/22/nexus---vector-and-matrix-maths-library).

## How?

Meshellator is quite easy to use. After you've added a reference to `Meshellator.dll` (or `Meshellator.Silverlight.dll`), the following line is enough to load from a file into a `Scene` object. The `Scene` object contains collections for meshes and materials, and should be quite intuitive to navigate.

``` csharp
Scene scene = MeshellatorLoader.ImportFromFile("85-nissan-fairlady.3ds");
```

Loading one of the built-in primitive shapes is just as easy. Each of the primitive shape methods have some parameters for size and tessellation level.

``` csharp
Scene scene = MeshellatorLoader.CreateFromTeapot(10, 20);
```

## Why?

Given that there's several open source asset import libraries already - in particular [Open Asset Import Library (or Assimp)](http://assimp.sourceforge.net/) - why make another one? Good question - if you're working purely on Windows, and don't mind a bit of interop into native code, then Assimp is indeed the solution I'd recommend. It's mature, well supported, and supports a whole host of formats.

Meshellator, on the other hand, is new and only supports 2 or 3 file formats at the moment. However, I believe it has a useful place in the .NET ecosystem, as the first (as far as I know) fully managed open source asset import library. Because it's fully managed, it also works on Silverlight.

## Where?

You can browse the Meshellator source code [here](https://github.com/tgjones/meshellator), and you can download the binaries from two places:

* [GitHub](https://github.com/tgjones/meshellator/downloads)
* [NuGet](http://nuget.org/Packages/Packages/Details/Meshellator-1-0-0-0)

## Viewer

The source code (but not the binary downloads) includes a viewer application.

![](/assets/posts/meshellatorviewer1.jpg)

The viewer application is not developed enough to be that useful, but it may be of technical interest - it combines [AvalonDock](http://avalondock.codeplex.com/), [FluentRibbon](http://fluent.codeplex.com/) and [SlimDX](http://slimdx.org/) in a WPF MVVM application based on [Caliburn](http://caliburn.codeplex.com/). As a historical aside, this viewer application is what eventually became [XBuilder](/blog/archive/2010/12/09/xbuilder-v-released), after I realised that the world didn't need another general-purpose model viewer, and the "model viewer as a Visual Studio extension" niche was more interesting.