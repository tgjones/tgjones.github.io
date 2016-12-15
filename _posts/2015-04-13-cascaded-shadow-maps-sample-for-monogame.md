---
title:   "Cascaded shadow maps sample for MonoGame"
date:    2015-04-13 16:25:00 UTC
---

## TL/DR

The source code for this sample is on GitHub, **but see the note in the next paragraph**.

* [Cascaded shadow maps sample for MonoGame - source code](https://github.com/tgjones/monogame-samples/tree/master/CascadedShadowMaps)

*Because this sample uses features that were added very recently to MonoGame (since v3.3), **you'll need to install a development build of MonoGame if you want to try it out**. Go to the [MonoGame downloads page](http://www.monogame.net/downloads/), and under the "Development Builds" heading, download the "MonoGame for Visual Studio" installer. (Of course, if you're reading this after v3.4 is released, then ignore this paragraph.)*

## The details

This is a sample I've been working on for MonoGame, demonstrating [cascaded shadow maps](https://msdn.microsoft.com/en-us/library/windows/desktop/ee416307%28v=vs.85%29.aspx). Cascaded shadow maps are great for rendering outdoor scenes, such as large terrains. Cascaded shadow maps are a variant of traditional shadow mapping, where you divide up the rendered scene into 2 or more parts, at increasing distances from the camera. The closest cascade is the highest quality, but only covers a small area, and the farthest cascade is the lowest quality, and covers a large area.

Cascaded shadow maps were first written about in 2007 (at least, Wolfgang Engel's article in ShaderX5 was the first to use that term, I believe). Since then, the basic algorithm has been improved in a number of ways. This sample includes several of those improvements - such as stabilisation, optimised PCF filtering, and normal offset in addition to a basic depth bias.

It is essentially a port of [MJP's shadows sample](https://mynameismjp.wordpress.com/2013/09/10/shadow-maps/). He did all the hard work! Go read that blog post for much more detail about the techniques demonstrated here.

Here are a couple of screenshots, the first showing just the shadows, and the second showing the cascades in different colours. The closest cascade is red, but isn't visible in this screenshot.

![](/assets/posts/shadow-mapping-1.png)

![](/assets/posts/shadow-mapping-2.png)

The only filtering option I have implemented is [the PCF technique used in The Witness](http://the-witness.net/news/2013/09/shadow-mapping-summary-part-1/). This technique looks pretty good, especially compared to other techniques that use a similar number of shader instructions. In the sample, you can toggle the filter size using `f`, from 2x2 all the way up to 7x7.

The sample allows you to play with both depth bias (little `b` to decrement, capital `B` to increment) and [normal offset](http://www.dissidentlogic.com/old/) (little `o`, capital `O`), so you can easily see the effect these have.

You can visualise the cascades with the `v` key, enable filtering across cascades with `k`, and stabilise cascades with `c`. Stabilising the cascades means that shadows won't shimmer as you move and rotate the camera. You do lose some quality by doing that, but the trade-off is usually worth it.

Hold the right mouse button to look around, and use the standard WASD keys to move around.

## Standing on the shoulders of giants

Graphics APIs (Direct3D, OpenGL) have evolved in the last few years to include optimisations either specifically intended for shadow maps, or directly applicable to shadow maps. These optimisations are commonly used in games nowadays, but those of us using XNA didn't have access to them. The beauty of open source is that with MonoGame, we can fix that. I wanted to make this sample as up-to-date as possible, so while I was working on this shadow mapping sample, I submitted a few pull requests to MonoGame. Fortunately they were accepted, and so these features are both used in this sample, and are now a part of MonoGame:

* [Comparison samplers](https://github.com/mono/MonoGame/pull/3628) - graphics hardware has supported hardware PCF filtering for a while now. In Direct3D, this feature was exposed through comparison samplers. XNA didn't support them, but MonoGame now does. (Unfortunately both this feature, and texture arrays, only work for Direct3D at the moment. That's because MojoShader, which MonoGame uses to translate HLSL shaders into GLSL, can't cope with comparison samplers or texture arrays. Hopefully that will be fixed soon. But right now, this limitation means that this sample only works for Direct3D on Windows.)

* [Texture arrays](https://github.com/mono/MonoGame/pull/3654) - cascaded shadow maps are normally rendered using 4 cascades, where each cascade is a separate shadow map. In XNA, you could either render these as 4 separate textures (slower, but simpler), or fit them all into 1 texture and manage the offset and scaling yourself (faster, but more complex). Texture arrays give you the best of both.

* [Depth clamping](https://github.com/mono/MonoGame/pull/3706) - disables clipping to the near and far planes. Without this, you have to back up the shadow caster camera the right distance so that you don't clip any of the scene behind the near plane or beyond the far plane. With depth clamping, you don't need to do that calculation.

I'm publishing this sample in the hope that it's useful to others. Some of the code is a bit tricky to get right, so if you want to add cascaded shadow maps to your game, hopefully this will help you get started. Feel free to ask questions in the comments below.

In case anyone's wondering what happened to the game engine I [teased](https://twitter.com/roastedamoeba/status/569886323462447104) on Twitter a couple of months ago - it's coming! But I found that a few components - in particular, shadows and terrain - were quite difficult to get working robustly. So I'm breaking them out into little sample apps, this one being the first, as a way to help me figure things out.