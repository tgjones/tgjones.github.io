---
title:   "3DS, OBJ, and NFF content importers for XNA 4.0"
date:    2010-11-20 00:09:00 UTC
---

Continuing my recent hobby of [digging up old projects](/blog/archive/2010/11/13/introducing-stitchup-generating-shaders-from-hlsl-shader-fragments) and [releasing them](/blog/archive/2010/11/16/an-xna--content-pipeline-processor-for-terrain), I want to mention a couple of content importers that I have been working on.

I'd like to preface this by saying that I've acted as an aggregator; most of this code was ported from somebody else's work, as I have acknowledged below. If there are any bugs, they are almost certainly my fault, and if you find one please [log an issue](https://github.com/tgjones/osiris/issues).

So the 3 content importers I've added to [Osiris](https://github.com/tgjones/osiris) (my catch-all project for XNA content pipeline extensions, including the [terrain geomipmapping processor](/blog/archive/2010/11/16/an-xna--content-pipeline-processor-for-terrain)) are:

* *3DS* - Lots of free 3D models on the web are in this format. My code was ported from [MRI3DS](http://www.pixelnerve.com/processing/libraries/mri3ds/), a Java 3DS importer.
* *OBJ* - Again, this is a common format. My code was ported from [OBJ_IMPORT](http://www.pixelnerve.com/processing/libraries/objimport/), a Java OBJ importer. I'm aware that there is an [OBJ importer on App Hub](http://create.msdn.com/en-us/education/catalog/sample/custom_model_importer), that may well be better than this one - the more choice the better, right?
* *NFF* - Actually, it's generous to call this an [NFF](http://tog.acm.org/resources/SPD/NFF.TXT) importer. All I really wanted was a simple file format to create primitives, and NFF was the closest I could find. I have implemented a tiny amount of the specification, and then bolted on my own additions. Have a look in the [ModelsDemo content project](https://github.com/tgjones/osiris/tree/master/src/ModelsDemo/ModelsDemoContent/Primitives/) for some examples. Just because it will break up the monotony of this post, I'll include one of the NFF files here:

``` c
# A material
f 0.3 0.3 0.3 0.8 0.2 16 0

# A plane at 0 0 0 with a 'width' of 50 and 'height' of 40
x-plane 0 0 0 50 40

# A red material
f 0.8 0.1 0.1 0.8 0.2 16 0

# A cube at -50 0 0 with a 'size' of 50
x-cube -50 0 0 50

# A green material
f 0.1 0.8 0.1 0.8 0.2 16 0

# A teapot at 50 0 0 with a 'size' of 50, with a tessellation level of 10
tess 10 x-teapot 50 0 0 50
```

The available primitives are:

* Teapot (every good primitive library needs one of these)
* Cube
* Sphere
* Cylinder
* Torus
* Plane

And before you ask, yes, the code for these is adapted from the [Primitives3D sample](http://create.msdn.com/en-US/education/catalog/sample/primitives_3d) on App Hub. Like I said, most of what I've done with these importers is package up other people's work, in a hopefully easy to use way.

As with all the software in my recent posts, this code is open source and [hosted on GitHub](https://github.com/tgjones/osiris). If you dig through the code, you might notice that it calls out to two other libraries called [Meshellator](https://github.com/tgjones/meshellator) and [Nexus](https://github.com/tgjones/nexus). These are two other open source projects I'm working on, but I'm not ready to talk about them in any great detail. The reason I've abstracted the mesh importing away from XNA is that I use the same library for a nascent software rasterizer.

On a tangential note, I have three reasons for releasing this software as open source:

* I'm legally obligated to by the licenses of the original code I ported from.
* I strongly believe in doing my bit to help us all, as the XNA community, to move beyond this kind of entry-level functionality and make some [cool games](http://www.indiegames-uprising.com/). It makes me happy to know that there are [many other people](http://www.google.com/search)?q=open+source+xna&ie=utf-8&oe=utf-8&aq=t&rls=org.mozilla:en-GB:official&client=firefox-a doing the same thing.
* In the past I have been one of the people who, when asked if they are going to release something as open source, have answered "yes, I want to, but I'm just going to polish it a little bit more", and then you never hear from them again. No longer - so what I'm releasing may not be bug-free, but I will respond to all comments and logged issues, and hey, it's open source.