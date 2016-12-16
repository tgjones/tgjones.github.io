---
title:   "Announcing XBuilder, a free & open source content viewer for XNA 4.0 integrated with Visual Studio 2010"
date:    2010-11-26 03:35:00 UTC
---

Following [two](/blog/archive/2010/11/20/xna-model-viewer-inside-visual-studio) [changes of name](/blog/archive/2010/11/23/a-sneak-preview-of-xna-inspector), I am now calling this project [XBuilder](https://github.com/tgjones/xbuilder). Why the change(s) of name? Well, I remembered the XNA Magic / Blade3D name change, and I didn't want to get into any legal trouble, especially since this is a non-money-making piece of software.

It's rather a grandiose name for something that boils down to a single tool window - but I have plans for a v0.2...

## The [TL;DR](http://en.wikipedia.org/wiki/Wikipedia:Too_long;_didn%27t_read) version

* [XBuilder](https://github.com/tgjones/xbuilder) is a Visual Studio 2010 extension which helps with XNA 4.0 development.
* XBuilder provides a "Content Preview" window, which can render your models and textures.
* In the Content Preview window, you can rotate your models, and set solid / wireframe fill mode.
* XBuilder uses whatever content importers and processors you have referenced from your content project. So if it builds in your content project, it should work with XBuilder.
* The source code is [available on github](https://github.com/tgjones/xbuilder).
* I have released v0.1, which you can [download here](https://github.com/tgjones/xbuilder/downloads).
* To install just double-click the .vsix file, and it should install into Visual Studio.

I plan to make it available online through Extension Manager, but I'd rather get some feedback first and make sure it's stable before I do that.

## Screenshots

Everybody likes screenshots, so here's a few.

There are two ways to open the Content Preview window. The first is the traditional View / Other Windows / Content Preview menu item:

![](/assets/posts/xbuilder1.jpg)

The second is a context menu item on supported file formats (more on that later):

![](/assets/posts/xbuilder2.jpg)

After clicking Preview on a model, the model will load:

![](/assets/posts/xbuilder3.jpg)

You can change it to wireframe:

![](/assets/posts/xbuilder4.jpg)

You can rotate it by dragging your mouse inside the window (on second thoughts you don't need a screenshot of that).

If the content importer or processor throws an error for any reason, it will appear in the Output window:

![](/assets/posts/xbuilder5.jpg)

Finally, it comes with an options page, which lets you configure the file extensions you want to activate the "Preview" context menu for. At the moment, XBuilder is not able to work out which files can be processed with the XNA Content Pipeline, so you need to tell it here. By default it will support the file types that XNA supports, but if you have your own importers you can include those file types here.

![](/assets/posts/xbuilder6.jpg)

## System requirements

Unfortunately 3rd party Visual Studio extensions don't work with Visual Studio Express editions. So you'll need a full-blown edition of Visual Studio (I'm actually not sure if Standard will work - perhaps somebody who has Standard can let me know).

You'll need XNA 4.0 installed.

## The future

There's a lot that could be done with this - and I have specific plans for a v0.2. Whether I actually build v0.2 depends very much on the response to this first release - I'm quite curious whether this tool will be useful for other people. If not, at least I had fun building it!

I learnt quite a bit about Visual Studio extensibility while building XBuilder, so if there's any interest, I could write a blog post or two about my experiences and the stumbling blocks I hit.

So, go [download](https://github.com/tgjones/xbuilder/downloads) it, have a play, and let me know what you think by leaving a comment. Actual bugs would be better on the [issues page](https://github.com/tgjones/xbuilder/issues).