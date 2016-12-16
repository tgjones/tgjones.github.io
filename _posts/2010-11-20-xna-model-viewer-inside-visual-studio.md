---
title:   "XNA Model Viewer inside Visual Studio 2010"
date:    2010-11-20 14:47:00 UTC
---

This is a work-in-progress - I just had the idea this morning to embed an XNA model viewer in a Visual Studio tool window. I had previously been working on a standalone model viewer, but then I thought, why recreate some of Visual Studio when you can use the real thing?

The current entry point is to right-click on a model file in a Content project, and click a "View Model" context menu item, but this could be changed so that if the model viewer tool window is open, it will update whenever you select a new model file.

Right now it only loads the model, but I could add some more tools to change the effect, change effect parameters, etc.

![image](/assets/posts/modelviewer1small.jpg)

What do you think? It's actually surprisingly easy to write a tool window for Visual Studio - so (armed with the Windows Forms samples from App Hub) this whole thing only took an hour. I like projects like that...