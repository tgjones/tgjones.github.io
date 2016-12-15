---
title:   "Creating thumbnails of a PDF in ASP.NET"
date:    2010-11-04 08:50:00 UTC
---

Recently I pushed a new feature to [my fork](https://github.com/tgjones/dynamic-image) of [DynamicImage](https://github.com/sitdap/dynamic-image).

## DynamicImage

Since you might not have heard of DynamicImage before, here's a brief introduction: DynamicImage is an open source image manipulation library for ASP.NET from [Sound in Theory](http://www.sitdap.com).

Have a look at the [site](http://dynamicimage.apphb.com) for information on getting started with DynamicImage.

DynamicImage composes images using two key concepts:

* *Layers*. An image can be composed of several layers. There are several layer types built-in: TextLayer, ImageLayer, RectangleShapeLayer and PolygonShapeLayer.
* *Filters*. Each layer can have zero or more filters applied to it. There are more than 15 filters built-in, including basic ones such as resize, crop, and border, and more advanced ones such as gaussian blur, distort corners, feather, shiny floor and curves.

It's easy to add both new layer types and new filter types.

## PDF Thumbnails

Getting back to the topic of this post, I added a new layer type, `PdfLayer`. Here it is in action:

``` xml
<sitdap:DynamicImage runat="server" ImageFormat="Jpeg">
	<Layers>
		<sitdap:PdfLayer SourceFileName="~/Assets/PDFs/pdf-test.pdf" PageNumber="1">
			<Filters>
				<sitdap:ResizeFilter Width="500" Mode="UseWidth" />
			</Filters>
		</sitdap:PdfLayer>
	</Layers>
</sitdap:DynamicImage>
```

You could also use the fluent interface if you're not using WebForms, or you prefer to build images in code:

``` csharp
string imageUrl = new DynamicImageBuilder()
	.WithLayer(
		new PdfLayerBuilder().SourceFileName("~/Assets/PDFs/pdf-test.pdf").PageNumber(1)
		.WithFilter(FilterBuilder.Resize.ToWidth(500))
	).Url;
```

Under the covers, this is using the excellent [GhostscriptSharp](https://github.com/mephraim/ghostscriptsharp) library from [Matthew Ephraim](http://www.mattephraim.com/).