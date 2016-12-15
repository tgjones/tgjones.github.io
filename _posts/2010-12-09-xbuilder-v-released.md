---
title:   "XBuilder v0.2 released"
date:    2010-12-09 00:27:00 UTC
---

## Cut to the chase...

XBuilder is an extension for Visual Studio 2010 which lets you preview your XNA assets (models, effects and textures) right inside Visual Studio. I have previously blogged about XBuilder [here](/blog/archive/2010/11/26/announcing-xbuilder-a-free--open-source-content), [here](/blog/archive/2010/12/01/xbuilder-updates---grid-rotation-indicator-mouse) and [here](/blog/archive/2010/12/01/xbuilder-updates---show-normals).

I've finished work on XBuilder v0.2, and it's available either using Visual Studio Extension Manager, or from the [Visual Studio Gallery](http://visualstudiogallery.msdn.microsoft.com/en-us/30e6bc7d-4d92-49da-ac40-adab43fd94a1).

XBuilder is open source, and the code is available in the [GitHub repository](https://github.com/formosatek/xbuilder).

## What's new since v0.1?

Quite a lot, as it turns out... I've blogged about some of these features before, but I'll include everything here anyway because, well, it pads out my blog post.

* Grid

You can customise the minor and major grid line spacing, as well as minor, major and axis grid colours.

* Rotation indicator

Click and drag this to rotate the camera. This is quite helpful for keeping track of what direction you're looking in.

![xbuilder image](/assets/520c9097f51f27a5a300001c/xbuilder8.jpg)

* Mouse wheel zoom

Pretty self-explanatory - when the content preview window has focus, use your mouse wheel to zoom in and out.

* Preview effects.

Effect preview is little more than a working prototype at the moment. You can preview effects and choose which primitive (sphere, cube, cylinder, torus, plane and of course teapot) to use to render the effect. If your effect includes parameters with semantics named `WORLDVIEWPROJECTION`, `WORLD`, `VIEW` or `PROJECTION`, XBuilder will set the matrices for you. If you use a custom `EffectProcessor` and your runtime effect class implements `IEffectMatrices`, XBuilder will set the matrices. But that's it - I need to implement DXSAS or something similar so that other parameters, such as lighting or material related parameters, can also be set.

* Uses your importer and processor settings

In v0.1, the importer was auto-detected by the XNA Content Pipeline and the processor was hard-coded, with no processor parameters. For v0.2, I've changed this to use whatever you have setup for in your content project. This was harder than I thought.

At first it looked easy - each file in the content project exposes an interface called `IVSBuildPropertyStorage`, which lets you access arbitrary MSBuild properties, such as `Importer` and `Processor`. However, you need to know the property name up-front - and for processor parameters, which have names like `ProcessorParameters_Scale`, this is a problem. `IVsBuildPropertyStorage` doesn't offer any way of getting a list of available property names.

My second attempt involved using the MSBuild API to open the content project file, find the relevant `ProjectItem` and read its properties. This worked... but only the first time you do it. MSBuild does something weird with caching opened projects in a static variable somewhere, so you can't open a project twice. I also tried opening the content project file as a standard XML file, but both of these approaches have the same root problem - even if they worked, they rely on you saving the content project file. If you're just tweaking processor parameter values, you don't want to save the project every time you want to update the preview.

My third, and successful, attempt was a bit different - I used a similar technique to what XNA uses internally. However, I didn't want to rely on reflection, as that's messy, so I stuck with public interfaces and types only. The end result is that XBuilder is able to pick up your processor parameter changes immediately, without you needing to save anything. You can tweak your parameter value, hit the keyboard shortcut (see below) and the preview will update.

* Bounding boxes

If you select this option, a bounding box will be drawn for each mesh within your model.

![](/assets/520c9096f51f27a5a300001a/xbuilder10.jpg)

* Solid+Wireframe shading mode

![](/assets/520c9096f51f27a5a300001b/xbuilder11.jpg)

Again, self-explanatory - I'm using something similar to [Catalin Zima's technique for wireframe without z-fighting](http://www.catalinzima.com/samples/12-months-12-samples-2008/drawing-wireframes-without-z-fighting/). You can configure the wireframe colour and z-bias values. Note that this shading mode will only work with models that use `BasicEffect`.

* Render normals

![](/assets/520c9099f51f27a5a3000020/xbuilder12.jpg)

Helpful for debugging - you can customise the length of the rendered normals as a proportion of the model bounding sphere radius.

* Alpha blending

If your model uses alpha blending, you'll want to enable this option. XBuilder will render opaque mesh parts first, followed by transparent mesh parts. At the moment it doesn't do any depth-sorting in the second pass.

* Keyboard shortcut

Ctrl+Shift+Alt+F10 will open the content preview window.

* More configurable options

![](/assets/520c9096f51f27a1dd000015/xbuilder9.jpg)

* Available from Visual Studio Extension Manager

![](/assets/520c9097f51f27a5a300001d/xbuilder13.jpg)

You can download and install XBuilder right within Visual Studio Extension Manager. When new versions are released, I'll be adding them to the gallery, so this is the route I'd recommend for installing XBuilder.

## What's next?

I'm going to take a bit of a break, and perhaps actually work on a game project... when I come back to XBuilder, I am going to clean up the code (which got a bit messy while working on v0.2), and then work on the [features I've got planned for v0.3](https://github.com/formosatek/xbuilder/issues). It's not too late to suggest features for v0.3 though - I'd really like this to become a community-driven project. (I know several people have asked for animation support - this is planned for v0.3.)

This is really just a hobby project for me, so it lives or dies on your feedback - please let me know what you think, if it's useful for you, what would make it useful for you, etc. If you find any bugs, please let me know [here](https://github.com/formosatek/xbuilder/issues).