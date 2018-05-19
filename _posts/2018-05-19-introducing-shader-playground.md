---
title:   "Introducing Shader Playground"
date:    2018-05-19 05:00:00 UTC
excerpt: "Introducing a new shader playground website, supporting compilation and transpilation of HLSL and GLSL"
tags:    shaders HLSL GLSL SPIR-V
---

**TL/DR: I've made a new website called [Shader Playground](http://shader-playground.timjones.io) that is sort of like [Compiler Explorer](https://godbolt.org/) but for shading languages. You can compile HLSL and GLSL using `fxc.exe`, `dxc.exe`, `glslangValidator.exe`, `spirv-cross.exe`, and `mali-sc.exe`. You can see the compiler output and disassembly. You can chain multiple shader compilers together to transpile one language to another. You can share permalinks to shaders.**

As you may know, I am very interested in shaders and shading languages ([exhibit A](/archive/2010/12/30/stitchup-in-action), [exhibit B](/archive/2014/01/07/introducing-hlslunit-unit-tests-for-your-hlsl-shader-code), [exhibit C](/archive/2014/11/24/getting-started-with-jetbrains-nitra), [exhibit D](/archive/2015/09/02/parsing-direct3d-shader-bytecode), [exhibit E](/archive/2015/10/06/introducing-hlsl-tools-for-visual-studio)). I frequently want to test little bits of shader syntax to see what the compiler does. That was why I wrote the [Try HLSL](/archive/2017/02/11/try-hlsl-online) website, first for myself and then I put it online last year. It's really very simple - type or paste HLSL code on the left, choose whether to compile with the old (fxc.exe) or new (dxc.exe) HLSL compilers, and see the disassembly (or compiler errors) on the right. It's been quite popular - I don't have stats running on it, but [my recent tweet](https://twitter.com/_tim_jones_/status/984034078788870144) about it was quite popular, so I think this is something that people find useful.

However, the Try HLSL website was always a bit limited. It only worked with HLSL, and it only worked with Microsoft's two HLSL compilers. Given the diversity of today's shader landscape (GLSL, SPIR-V, Metal), I wanted to make it better. So I did. I'm proud to launch Try HLSL's successor today:

* [Shader Playground](http://shader-playground.timjones.io).

{% responsive_image path: assets/posts/shader-playground.png alt: "Screenshot of Shader Playground website" %}

Let me first list the features:

* **Multiple input languages** (HLSL, GLSL, with the potential for more in the future)
* **Multiple compilers**:
  * `fxc.exe`
  * `dxc.exe`
  * `glslangValidator.exe`
  * `spirv-cross.exe`
  * `mali-sc.exe`
* **Compiler chaining**: create multiple compilation steps, so you can pipe HLSL to SPIR-V to Metal, for example.
* **Permalinks**: share links to a specific "playground", with input code and all the compilation steps saved

Now I'll go into more detail on some of these.

## Compilers

{% responsive_image path: assets/posts/shader-playground-compiler-options.png alt: "Screenshot of compiler dropdown" %}

The old Try HLSL site only supported Microsoft's old and new HLSL compilers, so with Shader Playground I wanted to expand that. You can now access these compilers. The exact ones shown in the dropdown will depend on the input language you have selected.

* [FXC](https://msdn.microsoft.com/en-us/library/windows/desktop/bb232919(v=vs.85).aspx) - `fxc.exe`, Microsoft'd legacy HLSL compiler
* [DXC](https://github.com/Microsoft/DirectXShaderCompiler) - `dxc.exe`, Microsoft's new open-source HLSL compiler
* [glslang](https://github.com/KhronosGroup/glslang) - `glslangValidator.exe`, Khronos's reference GLSL compiler that also now has an HLSL frontend
* [SPIRV-Cross](https://github.com/KhronosGroup/SPIRV-Cross) - `spirv-cross.exe`, Khronos's tool for disassembling SPIR-V back into high level languages (GLSL, HLSL, Metal, C++)
* [Mali offline compiler](https://developer.arm.com/products/software-development-tools/graphics-development-tools/mali-offline-compiler) - `mali-sc.exe`, ARM's offline compiler tool

## Compiler chaining

When a compiler outputs a language that can be fed into another compiler, you get an "Add Compiler" button. Click this, and you'll be able to chain together up to 5 compilers. Here's an example of compiling HLSL using DXC, outputting SPIR-V, piping that into SPIRV-Cross, and outputting GLSL:

{% responsive_image path: assets/posts/shader-playground-compiler-chaining.png alt: "Screenshot of compiler chaining" %}

Here are the current compilers with their input and output languages. I plan to add more to this list:

* FXC
  * Inputs:
    * HLSL
  * Outputs:
    * DXBC
* DXC
  * Inputs:
    * HLSL
  * Outputs:
    * DXIL
    * SPIR-V
* glslangValidator
  * Inputs:
    * GLSL
    * HLSL
  * Outputs:
    * SPIR-V
* SPIRV-Cross
  * Inputs:
    * SPIR-V
  * Outputs:
    * GLSL
    * Metal
    * HLSL
    * C++
* Mali offline compiler
  * Inputs:
    * GLSL
    * SPIR-V
  * Outputs:
    * [None]

You can use any valid combination of these, although right now that basically means using SPIRV-Cross at the end of the chain.

Some compilers have multiple outputs. When using DXC or glslang, for example, you can view the abstract syntax tree (AST) from the compiler:

{% responsive_image path: assets/posts/shader-playground-output-ast.png alt: "Screenshot of AST output" %}

With all compilers you can view the compiler output as well as the disassembly.

## Permalinks

Want to share a link to a shader? Once you've got everything setup (with whatever input code and compilers and parameters you want), click the "Share Permalink" button at the top-right:

{% responsive_image path: assets/posts/shader-playground-permalink.png alt: "Screenshot of Share Permalink button" %}

That will open a dialog, and after a short wait you'll get a URL that you can copy:

{% responsive_image path: assets/posts/shader-playground-permalink.png alt: "Screenshot of Share Permalink dialog" %}

## How it works

I wrote this site in C#, using ASP.NET Core. [The source code is up on GitHub](https://github.com/tgjones/shader-playground). It's hosted in Azure as an App Service (not on the free plan, for reasons, so if it gets popular I might start a Patreon...). In most cases I'm calling out to the backend shader compilers using `Process.Start(...)`, which is a bit scary because compiler options that you select on the site are passed as command line parameters to the backend compilers. I'm validating the input as best I can, and there's a timeout on running the process, but still... please don't break it.

I'm using GitHub Gists as the "database" for permalinks. There's an API rate limit that could theoretically be hit, but I think it will be fine. If it's not I'll have to think of something else.

## Future

Next, I plan to add more compilers:

* [Radeon GPU Analyzer (RGA)](https://github.com/GPUOpen-Tools/RGA)
* [HLSLcc](https://github.com/Unity-Technologies/HLSLcc)
* [PowerVR](https://community.imgtec.com/developers/powervr/tools/pvrshadereditor/)
* [GLSL optimizer](https://github.com/aras-p/glsl-optimizer)
* [hlsl2glslfork](https://github.com/aras-p/hlsl2glslfork)
* [HLSLParser](https://github.com/Thekla/hlslparser)
* [Slang](https://github.com/shader-slang/slang)

If you have a particular compiler that you'd like me to prioritise adding, please let me know.

Or if you want to dig in yourself, the whole Shader Playground website is [open source and on GitHub](https://github.com/tgjones/shader-playground). Contributions or feature suggestions are welcome! Bug reports are less welcome but still appreciated :)

And a grab bag of other things I'm thinking about:

* I'd also like to support multiple versions per compiler, so for example for `dxc.exe`, you'd be able to compare the outputs of multiple versions of that compiler, as it develops.
* I'd like to add support for Unity shaders, but [it's not at all straightforward](https://twitter.com/_tim_jones_/status/993813451276369921), if it's possible at all.
* The mobile version of the site is functional, but not ideal. My web dev knowledge is a bit out of date these days, so I need to brush up on flexbox in order to get things working how I want.

If you want an offline version of Shader Playground, I highly recommend looking at Josh Barczak's [Pyramid Shader Analyzer](https://github.com/jbarczak/Pyramid).I got a lot of inspiration from that project, so thank you Josh.