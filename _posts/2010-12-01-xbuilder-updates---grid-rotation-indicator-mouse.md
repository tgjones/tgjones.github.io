---
title:   "XBuilder updates - Grid, rotation indicator, mouse wheel zoom"
date:    2010-12-01 02:59:00 UTC
---

I did a little more work tonight on [XBuilder](https://github.com/tgjones/xbuilder). The next release will be v0.2, which is not yet ready ([here](https://github.com/formosatek/tgjones/issues) are some of the items I'm working on for v0.2, although I retain the right to add / remove from that list). However, I thought I'd share a video to hopefully whet your appetite for what's coming. This is all checked in to the [repo](https://github.com/tgjones/xbuilder), so if you want to live dangerously (and have the Visual Studio SDK installed), then feel free to grab the latest code and have a go.

The highlights are:

* *The preview window now shows a grid.* This should help you keep track of where you are. The major, minor, and axis line colours are all customisable.
* *You can zoom with the mouse wheel.*
* *You can rotate the camera using the cube in the top-right corner.* This is, err, inspired by a certain major 3D editing package, and I'm hoping it's okay for me to borrow the idea. If anybody thinks differently, please let me know.

Previously you could rotate only the object, not the camera. Right now you can only rotate the camera. I hope to add object rotation back in, but I'll wait till I have time to implement it properly.

<iframe width="560" height="315" src="https://www.youtube.com/embed/9fQnh88hKsM" frameborder="0" allowfullscreen></iframe>