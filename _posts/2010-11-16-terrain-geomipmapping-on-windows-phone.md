---
title:   "Terrain GeoMipMapping on Windows Phone 7"
date:    2010-11-16 08:58:00 UTC
---

*Update - 2010-11-16 17:10* [Lawrence Hodson](http://sharky.bluecog.co.nz/) has reported that [this works on an HTC Trophy](http://twitter.com/#!/SharkyNZ/status/4453810541957120) (pic included).

As I [promised in my last post](/blog/archive/2010/11/16/an-xna--content-pipeline-processor-for-terrain), I have made the necessary changes (which were not many) to get my terrain geomipmapping content pipeline processor working for Windows Phone 7. It turns out this is as easy as right-clicking the project in Visual Studio and selecting "Create Copy of Project for Windows Phone".

*I don't actually have a Windows Phone, so I haven't tested any of this on a real device. It's possible (probable) that performance could be an issue, and a smaller heightmap or higher "tau" value would need to be used.*

I have pushed the changes up to the [github repository](https://github.com/tgjones/osiris).

Some screenshots of the terrain demo running in the emulator:

![](/assets/posts/geomipmappingwp7-1.jpg)

![](/assets/posts/geomipmappingwp7-2.jpg)
