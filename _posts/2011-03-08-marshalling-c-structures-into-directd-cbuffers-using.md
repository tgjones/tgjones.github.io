---
title:   "Marshalling C# structures into Direct3D 11 cbuffers using SharpDX"
date:    2011-03-08 01:18:00 UTC
---

**UPDATE 08/03/2011**

* I have incorporated Alexandre Mutel's refactoring suggestion from his comment below, to avoid creating a `DataStream` every time.
* He also suggests an alternative approach, which is more elegant but requires more code.

It's tempting, after Googling a problem, finding no solutions, then going old-school and actually trying to solve it myself, and eventually coming up with a solution, to keep running and never look back. But this particular problem gave me lots of SEO-friendly keywords to put in the page title of a blog post, so here we are.

## Background

A bit of background - I'm working on some enhancements to [DotWarp](http://github.com/tgjones/dotwarp), which will allow an arbitrary number of directional lights, point lights, and spotlights to be applied to a scene. DotWarp uses [SharpDX](http://code.google.com/p/sharpdx/) to wrap Direct3D 11. DotWarp works with a single material type, defined in a file called `BasicEffect.fx`. I used that name because it roughly follows XNA's `BasicEffect` API (or rather it did until this most recent change to support arbitrary numbers of different light types).

The `.fx` extension is also misleading, because I'm not using the Direct3D 11 Effects framework - instead I compile the vertex and pixel shaders separately. I have a `BasicEffect` class defined in C# that tries to abstract that fact away from callers.

## The problem

Direct3D 11 shader parameters are declared inside `cbuffer` structures (parameters declared in global scope will be added to an implicit `cbuffer`). cbuffers themselves are outside the scope of this blog post, but they're pretty cool and offer some good performance improvements over the Direct3D 9 way of doing things. A typical `cbuffer` might look like this:

``` c
cbuffer BasicEffectVertexConstants : register(b0)
{
	matrix WorldViewProjection;
	matrix World;
}
```

The difficulty comes in setting the values for `WorldViewProjection` and `World` from your C# code. It gets even tricker if you want to use arrays inside your structures.

## A solution

What follows is fairly SharpDX-specific, although the same general principle should work for SlimDX too. First, we need to define a C# struct that matches the HLSL struct.

``` csharp
[StructLayout(LayoutKind.Explicit, Size = 128)]
internal struct BasicEffectVertexConstants
{
	[FieldOffset(0)]
	public Matrix WorldViewProjection;

	[FieldOffset(64)]
	public Matrix World;
}
```

You will notice I've used attributes to explicitly set the size and field offsets. This is to avoid differences between .NET and HLSL in how fields are packed. MSDN has a good page on [Packing Rules for Constant Variables](http://msdn.microsoft.com/en-us/library/bb509632.aspx), which covers the HLSL side. For the struct above, it would be packed correctly for HLSL even without the attributes, but that isn't always true, so I prefer to be consistent and always set the size and field offsets explicitly.

Now we need a couple of helper methods. The first is going to create a Direct3D 11 constant buffer resource, for a particular struct type. The second helper method is going to update that Direct3D 11 constant buffer resource with new values. *UPDATE* To keep things simple, we'll create a new class:

``` csharp
internal class ConstantBuffer<T> : IDisposable
	where T : struct
{
	private readonly Device _device;
	private readonly Buffer _buffer;
	private readonly DataStream _dataStream;

	public Buffer Buffer
	{
		get { return _buffer; }
	}

	public ConstantBuffer(Device device)
	{
		_device = device;

		// If no specific marshalling is needed, can use
		// SharpDX.Utilities.SizeOf<T>() for better performance.
		int size = Marshal.SizeOf(typeof (T));

		_buffer = new Buffer(device, new BufferDescription
		{
			Usage = ResourceUsage.Default,
			BindFlags = BindFlags.ConstantBuffer,
			SizeInBytes = size,
			CpuAccessFlags = CpuAccessFlags.None,
			OptionFlags = ResourceOptionFlags.None,
			StructureByteStride = 0
		});

		_dataStream = new DataStream(size, true, true);
	}

	public void UpdateValue(T value)
	{
		// If no specific marshalling is needed, can use 
		// dataStream.Write(value) for better performance.
		Marshal.StructureToPtr(value, _dataStream.DataPointer, false);

		var dataBox = new DataBox(0, 0, _dataStream);
		_device.ImmediateContext.UpdateSubresource(dataBox, _buffer, 0);
	}

	public void Dispose()
	{
		if (_dataStream != null)
			_dataStream.Dispose();
		if (_buffer != null)
			_buffer.Dispose();
	}
}
```

Armed with this class, we can start to write our actual code. First, we'll create the constant buffer. This should be in your initialisation code:

``` csharp
_vertexConstantBuffer = new ConstantBuffer<BasicEffectVertexConstants>(device);
```

Don't forget to `Dispose()` of that instance in your cleanup code.

Finally, we can update the values inside the `cbuffer` using the values from our C# struct with this code. Obviously you'll want to replace the structure type, and the setting of the field values, with your own parameters.

``` csharp
var vertexConstants = new BasicEffectVertexConstants();

Matrix wvp = ConversionUtility.ToSharpDXMatrix(World * View * Projection);
vertexConstants.WorldViewProjection = Matrix.Transpose(wvp);
vertexConstants.World = Matrix.Transpose(ConversionUtility.ToSharpDXMatrix(World));

_vertexConstantBuffer.UpdateValue(vertexConstants);

DeviceContext.VertexShader.SetConstantBuffer(0, _vertexConstantBuffer.Buffer);
```

## Field offsets

If your `cbuffer` is more complicated, then you'll have some other problems. Take this example:

``` c
cbuffer BasicEffectPixelConstants : register(b0)
{
	float3 CameraPosition = float3(0, 5, 20);

	bool LightingEnabled = true;

	float3 AmbientLightColor = float3(0.3, 0.3, 0.3);

	float3 DiffuseColor = float3(0.1, 0.7, 0.1);
	float3 SpecularColor = float3(1, 1, 1);
	float SpecularPower = 16;

	bool TextureEnabled = false;

	float Alpha = 1;
}
```

If you tried to map that onto a C# struct without explicitly setting field offsets, it wouldn't work, because HLSL has very strict [rules](http://msdn.microsoft.com/en-us/library/bb509632.aspx) for packing constant variables, which are not the same as .NET. I haven't found an automatic way of calculating the correct field offsets, but I have setup unit tests that allow me to check that I got it right. In this case, the corresponding C# struct looks like this:

``` csharp
[StructLayout(LayoutKind.Explicit, Size = 80)]
internal struct BasicEffectPixelConstants
{
	[FieldOffset(0)]
	public Vector3 CameraPosition;

	[FieldOffset(12)]
	public bool LightingEnabled;

	[FieldOffset(16)]
	public Vector3 AmbientLightColor;

	[FieldOffset(32)]
	public Vector3 DiffuseColor;

	[FieldOffset(48)]
	public Vector3 SpecularColor;

	[FieldOffset(60)]
	public float SpecularPower;

	[FieldOffset(64)]
	public bool TextureEnabled;

	[FieldOffset(68)]
	public float Alpha;
}
```

## Arrays

Finally, I can cover the case that prompted this blog post: using arrays inside your structs. You'll want to do this, for example, if you want to have a collection of lights, and loop through them in your shader code.

I have a DirectionalLight structure in HLSL:

``` c
struct DirectionalLight
{
	bool Enabled;
	float3 Direction;
	float3 Color;
};
```

... for which I've defined the corresponding structure in C#:

``` csharp
[StructLayout(LayoutKind.Explicit, Size = 32)]
internal struct BasicEffectDirectionalLight
{
	[FieldOffset(0)]
	public bool Enabled;

	[FieldOffset(4)]
	public Vector3 Direction;

	[FieldOffset(16)]
	public Vector3 Color;
}
```

I then have this `cbuffer`:

``` c
#define MAX_LIGHTS 16

cbuffer LightConstants : register(b1)
{
	int ActiveDirectionalLights;
	DirectionalLight DirectionalLights[MAX_LIGHTS];
}
```

... which maps to this C# structure:

``` csharp

private const int MaxLights = 16;

[StructLayout(LayoutKind.Explicit, Size = 4 + (32 * MaxLights) + 12 /* padding */)]
internal struct BasicEffectLightConstants
{
	[FieldOffset(0)]
	public int ActiveDirectionalLights;

	[FieldOffset(16), MarshalAs(UnmanagedType.ByValArray, SizeConst = MaxLights)]
	public BasicEffectDirectionalLight[] DirectionalLights;
}
```

The additional `MarshalAs` attribute on the array is the key to getting this to work. Without that, .NET won't know how to map the structure into the fixed size bit of memory that Direct3D has allocated to the `cbuffer`.

In summary - mapping structures from C# to HLSL cbuffers is non-trivial, but can certainly be done in entirely managed code. Have a look at the source code for [DotWarp](http://github.com/tgjones/dotwarp) to see a complete working example.