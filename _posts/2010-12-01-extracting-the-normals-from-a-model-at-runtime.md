---
title:   "Extracting the normals from a Model at runtime"
date:    2010-12-01 16:59:00 UTC
---

In case this is useful to somebody: the following code will extract the normals from a `ModelMeshPart` at runtime. Usually this is best done with a custom `ModelProcessor`, but that's not always possible.

``` csharp
private static Vector3[] GetNormals(ModelMeshPart meshPart)
{
	if (meshPart.VertexBuffer == null)
		return null;

	VertexBuffer vb = meshPart.VertexBuffer;
	if (meshPart.VertexBuffer.Tag != null) // Vertex buffers can be, and usually are, shared between ModelMeshPart's.
		return (Vector3[]) meshPart.VertexBuffer.Tag;

	VertexDeclaration vd = vb.VertexDeclaration;
	VertexElement[] elements = vd.GetVertexElements();

	Func<VertexElement,bool> normalElementPredicate = ve => ve.VertexElementUsage == VertexElementUsage.Normal && ve.VertexElementFormat == VertexElementFormat.Vector3;
	if (!elements.Any(normalElementPredicate))
		return null;

	VertexElement normalElement = elements.First(normalElementPredicate);

	Vector3[] normals = new Vector3[vb.VertexCount];
	vb.GetData(normalElement.Offset, normals, 0, vb.VertexCount, vd.VertexStride);

	return normals;
}
```

In fact, I'm not going to use this code, because I think I've found a way to use a custom `ModelProcessor` for my scenario, but perhaps it will be of use to somebody else.