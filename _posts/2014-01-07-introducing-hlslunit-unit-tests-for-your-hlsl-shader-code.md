---
title:   "Introducing HlslUnit: unit tests for your HLSL shader code"
date:    2014-01-07 15:54:00 UTC
excerpt: "HlslUnit is an open source .NET library that allows you to test your HLSL shaders in isolation, without running your game or application. Here I explain what it is and how to use it."
---

The *skip-to-the-end* version:

* HlslUnit is a .NET library that allows you to test your HLSL shaders without running your game or application.
* HlslUnit is an open source library. The [repository is hosted on GitHub](https://github.com/tgjones/slimshader).
* To get started, install the [HlslUnit NuGet package](http://www.nuget.org/packages/HlslUnit) into your test project.

### What does it do?

Whilst I'm not as good at Test-Driven Development ([TDD](http://en.wikipedia.org/wiki/Test-driven_development)) as I would like to be, I do try my best to provide some minimum level of test coverage for my C# code. But shaders (and I'm focusing here on HLSL shaders) are much more difficult to test in isolation. Normally, to "test" your shaders, you need to make a change to the HLSL code, start your game, and look around the 3D world until you're satisfied that you haven't broken anything. You could automate testing by having a known-good screenshot, and compare that against an automatically-taken screenshot - and this is a good idea, but it's not at all granular. That's more like an integration test - it tests your whole application. Ideally, you'd be able to test your HLSL code in isolation. That's where HlslUnit comes in - it allows you to test your HLSL code, and *only* your HLSL code.

HlslUnit works by providing a virtual machine that can execute your HLSL shaders on the CPU, without involving the GPU at all. It currently supports Shader Model 4.0 and Shader Model 5.0 shaders. HlslUnit provides a high-level `Shader` class that hides the underlying complexity behind a friendly API.

### Show me the code

Let's use a simple vertex shader as an example. Here's the HLSL code:

``` c
float4x4 World;
float4x4 WorldViewProjection;

struct VS_INPUT
{
  float4 Position  : POSITION;
  float3 Normal    : NORMAL;
  float2 TexCoords : TEXCOORD0;
};

struct VS_OUTPUT
{
  float4 Position  : SV_POSITION;
  float3 Normal    : TEXCOORD0;
  float2 TexCoords : TEXCOORD1;
};

VS_OUTPUT RenderSceneVS(VS_INPUT input)
{
  VS_OUTPUT output;   
  output.Position = mul(input.Position, WorldViewProjection);
  output.Normal = normalize(mul(input.Normal, (float3x3) World));
  output.TexCoords = input.TexCoords; 
  return output;    
}
```

When you compile this code, perhaps with `fxc.exe`, you'll get the following assembly code output. The assembly code is a human-readable version of the bytecode - HlslUnit works with the bytecode directly. Don't worry if this part doesn't make sense - it's only here for explanatory purposes, and can be safely ignored.

```
vs_2_0
dcl_position v0
dcl_normal v1
dcl_texcoord v2
dp4 oPos.x, v0, c0
dp4 oPos.y, v0, c1
dp4 oPos.z, v0, c2
dp4 oPos.w, v0, c3
dp3 r0.x, v1, c4
dp3 r0.y, v1, c5
dp3 r0.z, v1, c6
dp3 r0.w, r0, r0
rsq r0.w, r0.w
mul oT0.xyz, r0.w, r0
mov oT1.xy, v2
```

Next, we need to write some C# structs that match the HLSL structures. You might already have the vertex input structure - it's what you'd use to set the vertex data into a Direct3D vertex buffer. The `Matrix` and `Vector` types are from [SharpDX](http://sharpdx.org/), which I use and recommend - but HlslUnit itself is framework-agnostic.

``` csharp
[StructLayout(LayoutKind.Sequential)]
public struct ConstantBufferGlobals
{
  public Matrix World;
  public Matrix WorldViewProjection;
}

[StructLayout(LayoutKind.Sequential)]
public struct VertexShaderInput
{
  public Vector4 Position;
  public Vector3 Normal;
  public Vector2 TexCoords;
}

[StructLayout(LayoutKind.Sequential)]
public struct VertexShaderOutput
{
  public Vector4 Position;
  public Vector3 Normal;
  public Vector2 TexCoords;
}
```

Now, let's finally write the code to test this shader. I'm using [NUnit](http://www.nunit.org/) as the test framework here, but you can use your preferred test framework instead. (`ShaderTestUtility.CompileShader` is a simple method that wraps SharpDX's `ShaderBytecode.CompileFromFile` method. `$Globals` is the name assigned to the default constant buffer, which is used when you don't explicitly put your global variables into a constant buffer.)

``` csharp
[Test]
public void CanExecuteVertexShader()
{
  // Arrange.
  var shader = new Shader(ShaderTestUtility.CompileShader(
    "Shaders/VS/BasicHLSL.fx", "RenderSceneVS", "vs_4_0"));
  shader.SetConstantBuffer("$Globals", new VertexConstantBufferGlobals
  {
    World = Matrix.Identity,
    WorldViewProjection =
      Matrix.LookAtRH(Vector3.UnitZ, Vector3.Zero, Vector3.UnitY) *
      Matrix.PerspectiveFovRH(MathUtil.PiOverFour, 1, 1, 10)
  });
  var vertexInput = new VertexShaderInput
  {
    Position = new Vector4(3, 0, 2, 1),
    Normal = new Vector3(0, 1, 0),
    TexCoords = new Vector2(0, 1)
  };

  // Act.
  var output = shader.Execute<VertexShaderInput, VertexShaderOutput>(vertexInput);

  // Assert.
  Assert.That(output, Is.EqualTo(new VertexShaderOutput
  {
    Position = new Vector4(7.24264f, 0, -3.222222f, 1),
    Normal = new Vector3(0, 1, 0),
    TexCoords = new Vector2(0, 1)
  }));
}
```

HlslUnit's [unit tests](https://github.com/tgjones/slimshader/blob/master/src/HlslUnit.Tests/ShaderTests.cs) include a couple of further examples.

### How does it work?

HlslUnit is built on top of two other projects of mine, [SlimShader](https://github.com/tgjones/slimshader) and SlimShader.VirtualMachine. SlimShader is used to parse the Direct3D bytecode, and SlimShader.VirtualMachine is, as the name suggests, a virtual machine with an interpreter capable of executing the parsed bytecode. (There is a [JITter](https://github.com/tgjones/slimshader/blob/master/src/SlimShader.VirtualMachine.Jitter/JitShaderExecutor.cs) for SlimShader.VirtualMachine, which can execute HLSL shaders much more quickly than the interpreter, but it's overkill for unit tests, so I'm not using it for HlslUnit.)

I plan to write in more detail about how SlimShader and SlimShader.VirtualMachine work, but here's a brief explanation. When you compile your HLSL code, the Direct3D compiler generates bytecode - i.e. a sequence of raw bytes. It's unintelligible to humans, but meaningful once you know how the bytes are structured. SlimShader parses this sequence of bytes into a nice object-oriented structure, which includes (among other things) an array of `InstructionToken` objects. SlimShader.VirtualMachine has an interpreter that iterates through these instructions, and executes each one in turn, saving the output into a set of registers. HlslUnit pulls the output from these registers, casts it into the appropriate output structure, and returns it.

Everything happens on the CPU, in managed .NET code - in fact, HlslUnit is a [Portable Class Library](http://blogs.msdn.com/b/dotnet/archive/2013/10/14/portable-class-library-pcl-now-available-on-all-platforms.aspx), as are SlimShader and SlimShader.VirtualMachine, so they can be used on a number of different platforms.

[Take a look at the code](https://github.com/tgjones/slimshader) if you're interested in learning more.

### Getting started

By far the easiest way to get started is to install the [NuGet package](http://www.nuget.org/packages/HlslUnit). HlslUnit depends on the [SlimShader](http://www.nuget.org/packages/SlimShader) and [SlimShader.VirtualMachine](http://www.nuget.org/packages/SlimShader.VirtualMachine) packages, but that is all taken care of by the NuGet package system.

In a test method, create an instance of the `Shader` class. You'll need to pass it a `byte[]` array containing the compiled bytecode of your shader. (You can use SharpDX's `ShaderBytecode` class to compile your shader.)

Constant buffers are set with the `SetConstantBuffer<T>(string name, T value)` method. Make sure that your C# structure matches the one defined in HLSL.

Textures are slightly different - instead of passing a texture object, you pass a callback. Whenever your shader does a texture lookup, your callback will called with the texture coordinates requested by the shader. You can return whatever value you like - either a constant value, or vary it based on the coordinates. This makes it easier to write isolated tests - you don't need to use an actual texture.

After setting constant buffers and textures onto the `Shader` object, call the `Execute`, passing in your input structure. The return value is the shader output - you probably want to write some asserts at this point...

That's it!

### Over to you...

HlslUnit is already able to execute many HLSL shaders. It currently supports vertex and pixel shaders, for SM4.0 and SM5.0. It doesn't yet support every shader instruction - and that's where you come in. If HlslUnit interests you, and you try it out, please let me know if it breaks! The most likely explanation is that the interpreter doesn't support one of the instructions you've used in your shader. Please [log an issue](https://github.com/tgjones/slimshader/issues) on GitHub, and I'll take a look.

Unfortunately, XNA 4.0 only supports Shader Model 3.0, which uses an entirely different bytecode format. So at the moment, HlslUnit isn't compatible with XNA. [Fingers crossed for an XNA 5.0!](http://visualstudio.uservoice.com/forums/121579-visual-studio/suggestions/3725445-xna-5)