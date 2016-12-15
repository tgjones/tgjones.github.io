---
title:   "Building a virtual machine for HLSL shaders"
---

This is a continuation of my previous post on [parsing Direct3D shader bytecode](/blog/archive/2015/09/02/parsing-direct3d-shader-bytecode). If you haven't read that yet, I recommend doing so.

In this post, I'll look at through the creation of a virtual machine capable of executing HLSL shaders entirely on the CPU. First, we'll build an interpreter. Then we'll optimise that by building a just-in-time (JIT) compiler, which will be much faster than the interpreter. We'll do this in C#. It could be done in other languages, of course, but the Roslyn C# compiler comes in handy when it comes to just-in-time compiling HLSL to C#.

If you just want to see the source code, have a look at the [SlimShader](https://github.com/tgjones/slimshader) GitHub repository - specifically, the [virtual machine](https://github.com/tgjones/slimshader/tree/master/src/SlimShader.VirtualMachine) and [JITter](https://github.com/tgjones/slimshader/tree/master/src/SlimShader.VirtualMachine.Jitter).

## Declarations and instructions

In the [last post](/blog/archive/2015/09/02/parsing-direct3d-shader-bytecode), we took an HLSL shader like this:

``` c
struct VertexShaderInput
{
    float3 pos : POSITION;
    float2 tex : TEXCOORD;
};

struct PixelShaderInput
{
    float4 pos : SV_POSITION;
    float2 tex : TEXCOORD;
};

float4x4 WorldViewProjection;

PixelShaderInput VS(VertexShaderInput input)
{
    PixelShaderInput output;

    output.pos = mul(float4(input.pos, 1), WorldViewProjection);
    output.tex = input.tex;

    return output;
}
```

and ran it through Microsoft's `fxc.exe` HLSL compiler, producing a binary file that looks like this:

[![](/assets/55e41bfbf51f273003000008/standard/d3d-bytecode-binary-viewer.png)](/assets/55e41bfbf51f273003000008/d3d-bytecode-binary-viewer.png)

Then we disassembled this binary into its component chunks. One of the chunks is the `SHDR` chunk - as the name suggests, it contains the actual shader. If we parse the `SHDR` chunk for the example above, and then pretty-printed the results, we'd get this:

```
dcl_constantbuffer cb0[4], immediateIndexed
dcl_input v0.xyz
dcl_input v1.xy
dcl_output_siv o0.xyzw, position
dcl_output o1.xy
dcl_temps 1
mov r0.xyz, v0.xyzx
mov r0.w, l(1.000000)
dp4 o0.x, r0.xyzw, cb0[0].xyzw
dp4 o0.y, r0.xyzw, cb0[1].xyzw
dp4 o0.z, r0.xyzw, cb0[2].xyzw
dp4 o0.w, r0.xyzw, cb0[3].xyzw
mov o1.xy, v1.xyxx
ret
```

The first 6 lines, prefixed by `dcl_`, are declarations. The following 8 lines are instructions. So "all" we need to do are execute those instructions, somehow, and we're done! 

## Understanding the bytecode

Let's take a step back, and take some time to understand what each of those declarations and instructions actually means. (I gleaned this information partly from Direct3D header files, partly from other open source projects, and partly from trial and error.) We'll focus on this sample shader, in the interests of keeping this blog post to a reasonable length, but of course there are *many* more declaration and instruction types than these. [SlimShader](https://github.com/tgjones/slimshader) includes support for most of them.

Let's look at each in turn:



* HLSL used to use preshaders, which were executed on the CPU.
* WARP internally uses a virtual machine to execute shaders on the CPU. WARP contains a just-in-time compiler that converts HLSL bytecode into optimised assembly code.