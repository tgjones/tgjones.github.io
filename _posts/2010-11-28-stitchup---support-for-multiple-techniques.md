---
title:   "StitchUp - Support for multiple techniques"
date:    2010-11-28 01:49:00 UTC
---

I have just [pushed some code](https://github.com/tgjones/stitchup/commit/7351c20496c6db2007ac42b178212ac9d83d0b7e) and [added a new download](https://github.com/tgjones/stitchup/downloads) to [StitchUp](https://github.com/tgjones/stitchup).

This new version adds support for multiple techniques inside `.stitchedeffect` files. In case this is your first time reading about StitchUp, then a better place to start is [here](/blog/archive/2010/11/13/introducing-stitchup-generating-shaders-from-hlsl-shader-fragments).

This change involved quite a lot of refactoring, because it affects almost every part of the code generation process. The end result is that you can have multiple techniques that share some of the same fragments. The parameters and textures for those fragments will be declared only once, but the generated vertex and pixel shaders will be unique to each technique (or rather, to each pass, since you can have multiple passes inside each technique).

Because having multiple techniques in an effect will not be used by everybody, I didn't want to penalise those who don't need them by complicating the `.stitchedeffect` format. So I have left the original format intact, which means you can still declare your `.stitchedeffect` like this:

``` csharp
Fragments\VertexTypes\PositionNormalTexture.fragment
Fragments\VertexTypes\VertexPassThru.fragment
Fragments\BasicMaterial.fragment
Fragments\Lights\DirectionalLight.fragment
Fragments\LightingModels\Phong.fragment
Fragments\PixelColorOutput.fragment
```

If however, you want to use multiple techniques, you can use the new format (with the same file extension of `.stitchedeffect`):

``` c
stitchedeffect basic_effect;

[fragments]
fragment pnt = "Fragments\VertexTypes\PositionNormalTexture.fragment";
fragment vpt = "Fragments\VertexTypes\VertexPassThru.fragment";
fragment bm = "Fragments\BasicMaterial.fragment";
fragment dl = "Fragments\Lights\DirectionalLight.fragment";
fragment ph = "Fragments\LightingModels\Phong.fragment";
fragment pco = "Fragments\PixelColorOutput.fragment";

[techniques]
technique t1
{
	pass p1
	{
		fragments = [pnt, vpt, bm, dl, ph, pco];
	}
}

technique t2
{
	pass p1
	{
		fragments =
		[
			pnt, vpt, bm, dl, ph,
			"Fragments\AlternatePixelColorOutput.fragment"
		];
	}
}
```

As you'll notice, you can mix and match "declared" fragments with inline strings. But there is a difference between the two.

* If you declare a fragment (in the `[fragments]` block), then the StitchUp compiler will only generate parameter and texture declarations for that fragment once, regardless of how many times you reference that fragment in technique passes.
* If you use a string literal inside a `pass`, such as in the second technique above, the StitchUp compiler will assume that fragment is unique to that pass, and generate unique parameter and texture declarations for it.

I have updated the demo project to include a few examples of different `.stitchedeffect` files.

I hope some of you find this a useful addition. Let me know what you think - and as always, if you find a bug, please [let me know](https://github.com/tgjones/stitchup/issues).