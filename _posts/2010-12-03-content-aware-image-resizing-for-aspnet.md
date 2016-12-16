---
title:   "Content Aware Image Resizing for ASP.NET"
date:    2010-12-03 01:25:00 UTC
---

Content Aware Image Resizing (also known as [seam carving](http://en.wikipedia.org/wiki/Seam_carving)) is an algorithm, developed by Shai Avidan, which resizes an image by removing _seams_ (paths of least importance) to reduce image size, or inserting seams to extend it. This is nice because it allows you to crop images without missing important details.

I didn't know about this technique until a friend pointed me in the direction of [CAIR](http://sites.google.com/site/brainrecall/cair), an open source implementation of seam carving. Never one to turn a blind eye to unnecessary but cool software, I have added a `ContentAwareResizeFilter` to [my fork](https://github.com/tgjones/dynamic-image) of [DynamicImage](https://github.com/sitdap/dynamic-image) (DynamicImage is an open source image manipulation library for ASP.NET - there's some [getting started documentation here](https://github.com/sitdap/dynamic-image/wiki)). `ContentAwareResizeFilter` simply calls the CAIR executable, so the credit for all the hard work goes to [Brain Recall](http://sites.google.com/site/brainrecall).

An example might make things clearer. Here is the original image:

![](/assets/posts/dynamicimagecair1.jpg)

Here is the image reduced in width using scaling:

![](/assets/posts/dynamicimagecair2.jpg)

And here is the image reduced in width using content aware image resizing:

![](/assets/posts/dynamicimagecair3.jpg)

The DynamicImage markup for the final image above is:

``` xml
<sitdap:DynamicImage runat="server">
	<Layers>
		<sitdap:ImageLayer SourceFileName="~/Assets/Images/sunset.jpg">
			<Filters>
				<sitdap:ContentAwareResizeFilter Width="350" ConvolutionType="V1" />
			</Filters>
		</sitdap:ImageLayer>
	</Layers>
</sitdap:DynamicImage>
```

If you don't use web forms, you can use the fluent interface in code:

``` csharp
string imageUrl = new DynamicImageBuilder()
	.WithLayer(
		LayerBuilder.Image.SourceFile("~/Assets/Images/sunset.jpg")
		.WithFilter(FilterBuilder.ContentAwareResize.ToWidth(350))
	).Url;
```

Here's another example (you might recognise [this one](http://commons.wikimedia.org/wiki/File:Broadway_tower_edit.jpg) from the [Wikipedia article](http://en.wikipedia.org/wiki/Seam_carving), but the images below are created using DynamicImage). The original image:

![](/assets/posts/dynamicimagecair4.jpg)

Reduced in width using scaling:

![](/assets/posts/dynamicimagecair5.jpg)

Reduced in width using content aware image resizing:

![](/assets/posts/dynamicimagecair6.jpg)

In some situations, I think this effect is potentially very useful.

If image manipulation in ASP.NET interests you, you might be interested in my previous posts on [creating PDF thumbnails](/blog/archive/2010/11/04/creating-thumbnails-of-a-pdf-in-aspnet) and [creating website thumbnails](/blog/archive/2010/11/04/creating-website-thumbnail-images-in-aspnet).