---
title:   "New SharpDX module for Gemini - Direct3D in your AvalonDock"
date:    2014-01-10 17:43:00 UTC
excerpt: "Gemini now has a module for integrating SharpDX-rendered content into documents and tool panes."
---

I've just added a new module (and demo) to [Gemini](http://documentup.com/tgjones/gemini), which is designed to integrate [SharpDX](http://sharpdx.org)-rendered content into documents and tool panes. Specifically, I'm using SharpDX Toolkit, because it has a nice simple API, but you can still drop down to the lower-level API when you need to.

![](/assets/posts/gemini-sharpdx.png)

It's quite easy to use. In the XAML for your document or tool pane view, add the `DrawingSurface` control: 

``` xml
<gemini:DrawingSurface x:Name="GraphicsControl"
  LoadContent="OnGraphicsControlLoadContent"
  Draw="OnGraphicsControlDraw"
  UnloadContent="OnGraphicsControlUnloadContent" />
```

In the view code-behind, implement methods to load content, draw and unload content. Have a look at the [demo source code](https://github.com/tgjones/gemini/blob/master/src/Gemini.Demo.SharpDX/Modules/SceneViewer/Views/SceneView.xaml.cs) for an example. These methods will be called automatically when needed (i.e. when the device is lost, or the user drags a document out to a floating window, etc.).

I added this module because I want to write a few SharpDX samples, and I'd like to host them in a WPF application.

I can't take credit for most of the code. I started with the [WPFHost SharpDX Direct3D10 Sample](https://github.com/sharpdx/SharpDX/blob/master/Samples/Direct3D10/WPFHost), modified it to work with SharpDX Toolkit, and integrated it into Gemini.

You'll find the Gemini source code in the [Gemini repository on GitHub](https://github.com/tgjones/gemini).

I will release new NuGet packages, including one for this module, in due course. This SharpDX module joins the existing [XNA module](http://documentup.com/tgjones/gemini#what-modules-are-built-in/xna-module).