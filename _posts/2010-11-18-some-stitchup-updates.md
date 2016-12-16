---
title:   "Some StitchUp updates"
date:    2010-11-18 22:45:00 UTC
---

## The short version

There is a new build of [StitchUp](/blog/archive/2010/11/13/introducing-stitchup-generating-shaders-from-hlsl-shader-fragments) available [here](https://github.com/tgjones/stitchup/downloads).

It includes these new features:
* The parser supports initial values for parameters.
* The parser supports C-style single line comments.
* The build output window in Visual Studio now shows a clickable message for the generated effect 

## The long version

After my last post introducing [StitchUp](/blog/archive/2010/11/13/introducing-stitchup-generating-shaders-from-hlsl-shader-fragments), I received some feedback, including a very detailed comment from "Alejandro" (sorry Alejandro, I'm not implying I'm on first name terms with you, I just don't know your surname). I started to reply to his comment, but it got quite big, so I thought I'd write a new post, which will allow me to wax more lyrical about some new features. I also think some of his questions might be interesting for a wider audience.

So, here are his questions, and my responses:

> In the [params] body, how can I set a starting default value? like:
[params]
float4 color : DIFFUSE_COLOR; // = { 1,1,1,1}; ?

It wasn't possible before, but it is now. I've modified the parser to include support for initial values. So the following declarations are now possible - note that this is only valid within the `[params]` block, since they are the only variable declarations that initial values make sense on.

``` c
[params]
float3 lightDir : LIGHT_DIRECTION = { 1.0f, -1, 1 };
float3 lightDir = { 1.0f, -1, 1 };
float3 lightDir = float3(1.0f, -1, 1);
```

Next question:

> ... the BasicMaterial fragment doesn't import anything but has a [vertex] attribute and interpolator. Is there some magic wand going around behind the scenes? Like: "No one output-ed an uv and you need it, so I'll just give to you"?.

Yes, that is exactly what is happening. The code generator follows these rules:

* *If the fragment includes a vertex shader, then use it - don't auto-generate one.* You might have values in the `[interpolators]` block that you want to pass to the pixel shader, but that is not going to happen automatically. You need to use the `output` meta-function, which is used by `PositionNormalTexture.fragment` in the demo project.
* *If the fragment doesn't include a vertex shader, then auto-generate one.* The code generator creates pass-through statements for all values in the `[interpolators]` block. For BasicMaterial, the auto-generated vertex shader looks like this:

``` c
// -------- vertex shader basic_material2 --------
void basic_material2_vs(basic_material2_VERTEXINPUT input, inout VERTEXOUTPUT output)
{
	output.basic_material2.uv = input.uv;
}
```

Next question:

> Is it still possible to #include an .fxh with basic functions and methods? I tried but failed big time. Or the correct practice would be to think about "Helper Fragments" and you would be actually "importing the methods"? (for things like random, poisson things, etc).

Yes, this is supported, in two ways.

1. You can create a fragment with only a `[headercode]` block. See `DirectionalLight.fragment` in the demo project for an example. I think this is the most elegant solution.
2. You can use `#include` within an `__hlsl__` block, but because of the way I'm calling `EffectProcessor` with a dynamically generated effect, you'll need to use absolute path names (i.e. `C:\...\MyInclude.fxh`).

Next question:

> Is there a way to see the stitched .fx file (ouput somewhere in some folder). I only see it when there is an error (usually name mismatch). Visual studio keeps it in a temporal folder.

The output is always called `StitchedEffect.fx`, and it's always saved in your temp folder. You're right that Visual Studio only shows you a clickable error message if there's an error. I've actually [asked on the App Hub forums](http://forums.create.msdn.com/forums/t/66862.aspx) how to do this for informational messages. In the meantime, I have added a message to the build output log, which you can double-click on to open the stitched effect file. The message looks like this:

```
C:\Users\Tim Jones\AppData\Local\Temp\StitchedEffect.fx :	Stitched effect generated (double-click this message to view).
```

Next question:

> When assigning Visual C++ as the .fragment text editor (to get a little bit of syntax highlighting) commenting a line outside of the __hlsl__ body (via ctrl+e, c) will always crash Visual Studio. (Is there a way to assign NShader as the syntax highlighter for the .fragment? that would be awesome)

The first little issue is that the StitchUp parser didn't allow for comments. I've implemented that now, so you can use C-style single-line comments in your `.fragment` files:

``` c
float3 lightDir : LIGHT_DIRECTION = { 1.0f, -1, 1 }; // This is a comment.
//float3 lightDir : LIGHT_DIRECTION = { 1.0f, -1, 1 };
```

I hadn't heard of [NShader](http://nshader.codeplex.com/) before - that's a nice project. The only way I could see to add support for `.fragment` files was to change the original source code. Since the developers of NShader are unlikely to be interested in supporting StitchUp-specific extensions, I have [created a fork of NShader](https://github.com/tgjones/nshader-stitchup). You can [download the StitchUp-specific VSIX file](https://github.com/tgjones/nshader-stitchup/downloads). So far I have only added support for the `.fragment` extension, but it would be nice to add support for StitchUp keywords, such as `export`, `import` and `output`. Oh, and you probably don't want to install it at the same time as the official NShader package - in any case, Visual Studio probably won't let you, because I haven't changed of the package GUIDs.

After installing this custom version of NShader, your `.fragment` files should look like this:

![](/assets/posts/stitchup5.jpg)

Hope this helps - as always, please post thoughts, feedback, etc. below.