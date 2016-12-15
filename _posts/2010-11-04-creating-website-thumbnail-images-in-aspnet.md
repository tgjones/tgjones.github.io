---
title:   "Creating website thumbnail images in ASP.NET"
date:    2010-11-04 08:55:00 UTC
---

Following on from my post about [generating PDF thumbnails in ASP.NET](/blog/archive/2010/11/04/creating-thumbnails-of-a-pdf-in-aspnet), here is another one about generating website thumbnail images using [DynamicImage](https://github.com/sitdap/dynamic-image).

``` xml
<sitdap:DynamicImage runat="server" ImageFormat="Jpeg">
	<Layers>
		<sitdap:WebsiteScreenshotLayer WebsiteUrl="http://www.microsoft.com">
			<Filters>
				<sitdap:ResizeFilter Width="500" Mode="UseWidth" />
			</Filters>
		</sitdap:WebsiteScreenshotLayer>
	</Layers>
</sitdap:DynamicImage>
```

<p>You can also use the fluent interface:</p>

``` csharp
string imageUrl = new DynamicImageBuilder()
	.WithLayer(
		new WebsiteScreenshotLayerBuilder().WebsiteUrl("http://www.microsoft.com")
		.WithFilter(FilterBuilder.Resize.ToWidth(500))
	).Url;
```

The preceding code will result in this image:

![](/assets/520c9093f51f27a5a3000016/websitethumbnaillayer1.jpg)

As you can see, the image is of the whole web page, not just the cropped window you'd normally see. If you want that, you can use the `Crop` filter:

``` xml
<sitdap:DynamicImage runat="server" ImageFormat="Jpeg">
	<Layers>
		<sitdap:WebsiteScreenshotLayer WebsiteUrl="http://www.microsoft.com">
			<Filters>
				<sitdap:ResizeFilter Width="500" Mode="UseWidth" />
				<sitdap:CropFilter Width="500" Height="500" />
			</Filters>
		</sitdap:WebsiteScreenshotLayer>
	</Layers>
</sitdap:DynamicImage>
```

Now you should have:

![](/assets/520c9092f51f27a1dd000012/websitethumbnaillayer2.jpg)

You can optionally specify a timeout, after which if the screenshot capture process hasn't finished, it will terminate and not produce an image. 

``` xml
<sitdap:WebsiteScreenshotLayer WebsiteUrl="http://www.microsoft.com" Timeout="3000" />
```

``` csharp
new WebsiteScreenshotLayerBuilder()
	.WebsiteUrl("http://www.microsoft.com")
	.Timeout(3000)
```

Since these images will take a while to produce, you'll probably want to [enable caching](https://github.com/sitdap/dynamic-image/wiki/Getting-Started) in DynamicImage.

Thanks are due to to ['NoLoveLust's work on wrapping CutyCapt.exe](http://nolovelust.com/post/C-Website-Screenshot-Generator-AKA-Get-Screenshot-of-Webpage-With-Aspnet-C.aspx), which provided a starting point for this code.

To get it working, you'll need to copy `CutyCapt.exe` from the [Tools folder](https://github.com/tgjones/dynamic-image/tree/master/Tools/) in the repo to `App_Data/DynamicImage`.