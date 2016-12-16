---
title:   "Introducing StitchUp: \"Generating Shaders from HLSL Shader Fragments\" implemented in XNA 4.0"
date:    2010-11-13 14:01:00 UTC
---

_or "A stitch in time saves lots of mucking about with shader permutation explosion..."_ 

*Want to skip to the end?*

* The project is open source and hosted in a [github repository](https://github.com/tgjones/stitchup).
* You can download the latest release [from here](https://github.com/tgjones/stitchup/downloads).
* There's a [demo project](https://github.com/tgjones/stitchup/tree/master/src/StitchUp.Demo/) in the repo to get you started.

## A bit of background

HLSL shaders have been around for a while now. There are a multitude of different effects that you can create with shaders. They are very powerful, and taken on their own, fairly easy to work with. However, problems start to creep in when you're writing a game with several different shaders (what I'm really talking about are *effects*, which combine vertex and pixel shaders into a single compilable unit). As soon as you start working with multiple effects, you hit the usual problems:

* I want something like this shader, but with an additional detail texture instead of one.
* I want to add a new light, but I don't want to code it for each of my effects.
* I want to add a new _type_ of light, but I don't want to code it for each of my effects.
* etc.

This boils down to "how do I handle shader permutations?" Of course, there are *many* ways of solving this problem, ranging from hard-coded `#if` statements, through to fully-fledged node-based shader graphs.

## What does it do?

[StitchUp](https://github.com/tgjones/stitchup) is one possible solution of the problem of shader permutation explosion. It is an implementation, using an XNA 4.0 Content Pipeline extension, of Shawn Hargreaves' article [Generating Shaders From HLSL Fragments](http://www.talula.demon.co.uk/hlsl_fragments/hlsl_fragments.html). I see StitchUp as sitting somewhere in the middle between hard-coded `#if` statements and node-based shader graphs. It's far more flexible than preprocessor defines or includes, but it's less powerful (and also simpler to work with as a developer) than a shader node graph.

For much more detail and background on HLSL fragments, go [have a read of the article this project is based on](http://www.talula.demon.co.uk/hlsl_fragments/hlsl_fragments.html). I have implemented everything in the article except for the last section on "Adaptive Fragments". I have also changed the syntax used to define fragments, as you'll see below.

StitchUp allows you to create effects by defining individual shader fragments. These fragments are stitched together to form the complete effect. Fragments can include a vertex shader part, a pixel shader part, or both. They can specify that they require effect parameters, vertex attributes, interpolated attributes, and textures. Fragments can communicate with each other by exporting and importing named values.

## What does it look like?

Here's an example fragment:

``` c
fragment base_texture;

[textures]
texture2D color_map;

[vertexattributes]
float2 uv;

[interpolators]
float2 uv;

[ps 2_0]
__hlsl__
void main(INPUT input, inout OUTPUT output)
{
	output.color = tex2D(color_map, input.uv);
}
__hlsl__
```

Here's a simple vertex fragment. This fragment shows a bit more of the fragment definition syntax - it's pretty close to HLSL, including semantics.

``` c
fragment vertex_transform;

[parameters]
matrix wvp : WORLDVIEWPROJECTION;

[vertexattributes]
float3 position : POSITION;
float3 normal : NORMAL;

[interpolators]
float3 normal;

[vs 2_0]
__hlsl__
void main(INPUT input, inout OUTPUT output)
{
	output.position = mul(float4(input.position, 1), wvp);
	output(normal, input.normal);
}
__hlsl__
```

All variable names will get mangled in the final effect file, to ensure they are unique across all fragments. This happens behind the scenes, and you shouldn't need to worry about it (until something goes wrong and then you can [log an issue](https://github.com/tgjones/stitchup/issues)).

## How does it work?

StitchUp is implemented as an XNA 4.0 Content Pipeline extension. In your Content project, you have many `.fragment` files, and one or more `.stitchedeffect` files. These files are associated with StitchUp-specific importers and processors. At compile-time, StitchUp will take the fragments and compile them into an effect. From here on it's as though the effect came from the standard Effect Importer / Processor, and you can load it into your XNA game as you would any other effect.

StitchUp also comes with a custom Model Importer. This importer lets you specify which effect to use to render the model, and will build and load that effect instead of the standard BasicEffect.

## Can I see a demo?

Sure! There's a demo project in the [repository](https://github.com/tgjones/stitchup). This project is based on the [Chase Camera sample](http://create.msdn.com/en-US/education/catalog/sample/chasecamera) from [App Hub](http://create.msdn.com/). The fragments here combine to form a simplified version of BasicEffect (please forgive the nonsensical logic of this sentence).

The fragments are included in the demo content project like this:

![](/assets/posts/stitchup1.jpg)

The file properties for a `.fragment` file look like this:

![](/assets/posts/stitchup2.jpg)

The file properties for a `.stitchedeffect` file look like this:

![](/assets/posts/stitchup3.jpg)

The file properties for one of the models file look like this:

![](/assets/posts/stitchup4.jpg)

At runtime, the effect parameter values are set in the `DrawModel` method in `ChaseCameraGame.cs`. Because the parameter names for each fragment will get mangled to ensure they are unique, it's slightly awkward to refer to parameters by name (this is an area I'd like to improve in the future). So I generally use semantics, if I know I'll only use a fragment once per effect.

``` csharp
Matrix wvp = transforms[mesh.ParentBone.Index] * world * camera.View * camera.Projection;
effect.Parameters.GetParameterBySemantic("WORLD").SetValue(transforms[mesh.ParentBone.Index] * world);
effect.Parameters.GetParameterBySemantic("WORLDVIEWPROJECTION").SetValue(wvp);	
effect.Parameters.GetParameterBySemantic("CAMERA_POSITION").SetValue(camera.Position);

effect.Parameters.GetParameterBySemantic("AMBIENT_LIGHT_DIFFUSE_COLOR").SetValue(new Vector4(0.1f, 0.1f, 0.1f, 1.0f));
effect.Parameters.GetParameterBySemantic("LIGHT_DIRECTION").SetValue(Vector3.Normalize(new Vector3(0.2f, -0.9f, 0.2f)));
effect.Parameters.GetParameterBySemantic("LIGHT_DIFFUSE_COLOR").SetValue(new Vector4(0.5f, 0.6f, 0.4f, 1.0f));

effect.Parameters.GetParameterBySemantic("DIFFUSE_TEXTURE_ENABLED").SetValue(true);
effect.Parameters.GetParameterBySemantic("SPECULAR_COLOR").SetValue(Vector3.One);
effect.Parameters.GetParameterBySemantic("SPECULAR_POWER").SetValue(16);
effect.Parameters.GetParameterBySemantic("SPECULAR_INTENSITY").SetValue(1);
```

## Over to you...

If you do decide to give StitchUp a try, I'd appreciate any feedback you might have, positive or otherwise. Either leave a comment here or log an issue in the github repository. I'll gladly accept push requests for bug fixes.

## The future?

* There are some unit tests, but not enough. I'd like to expand on these.
* Documentation! The wiki is a bit empty at the moment...
* Adaptive fragments? This is the only part of the original article that I haven't implemented.
* Runtime classes to act as a wrapper for the compile-time fragments. They could keep track of the fragment names and make it a bit easier to set parameter values.
* Right now you need to have a fragment with a vertex shader to do the vertex transformation. StitchUp could generate this automatically.

## Acknowledgements

If it wasn't obvious already, this whole project owes its existence to Shawn Hargreaves' article [Generating Shaders From HLSL Fragments](http://www.talula.demon.co.uk/hlsl_fragments/hlsl_fragments.html), which first appeared in ShaderX3 and is now available from [his website](http://www.talula.demon.co.uk/). Shawn also helped with some technical questions I had.