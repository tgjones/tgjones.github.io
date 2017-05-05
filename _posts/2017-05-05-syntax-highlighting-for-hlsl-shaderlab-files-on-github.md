---
title:   "Syntax highlighting for HLSL and ShaderLab files on GitHub.com"
date:    2017-05-05 15:00:00 UTC
excerpt: "My pull requests to add syntax highlighting to HLSL and ShaderLab files on GitHub.com were merged."
tags:    HLSL ShaderLab GitHub HLSL-Tools
---

## Making the world prettier, one GitHub.com page at a time

I don't know about you, but one of the things I've always liked about GitHub is how nice it looks when you're browsing code files.

That is, unless you're looking at HLSL or Unity ShaderLab shaders, in which case you're stuck with plain, non-syntax-highlighted, text. (Or in the case of ShaderLab, GitHub pretended they were GLSL files, resulting in missing highlighting for all the ShaderLab-specific parts, and poor highlighting for `CGPROGRAM` blocks because Cg has more in common with HLSL than GLSL.)

No longer! My pull requests to [add syntax highlighting to HLSL](https://github.com/github/linguist/pull/3469) and [ShaderLab shaders](https://github.com/github/linguist/pull/3490) were merged.

Here are before / after screenshots of an HLSL file:

{% responsive_image path: assets/posts/github-hlsl-before.png alt: "HLSL syntax highlighting on github.com - Before" %}

{% responsive_image path: assets/posts/github-hlsl-after.png alt: "HLSL syntax highlighting on github.com - After" %}

And here are before / after screenshots of [a ShaderLab file](https://github.com/Unity-Technologies/PostProcessing/blob/9ec5df8e60806f226ed770931300f14ef4cf6638/PostProcessing/Resources/Shaders/DepthOfField.shader):

{% responsive_image path: assets/posts/github-shaderlab-before.png alt: "ShaderLab syntax highlighting on github.com - Before" %}

{% responsive_image path: assets/posts/github-shaderlab-after.png alt: "ShaderLab syntax highlighting on github.com - After" %}

## Shameless plug

If you're still reading, chances are that you work with either HLSL or ShaderLab shaders on a regular basis. So if you don't already use it, you might be interested in [HLSL Tools for Visual Studio](https://marketplace.visualstudio.com/items?itemName=TimGJones.HLSLToolsforVisualStudio). I've just released v1.1, with [many bug fixes and some nice improvements](https://github.com/tgjones/HlslTools/blob/master/CHANGELOG.md).

I'm hard at work on adding ShaderLab support, although there's still a lot to do - keep an eye out here on my blog or on [Twitter](https://twitter.com/_tim_jones_) for when I have something to announce.