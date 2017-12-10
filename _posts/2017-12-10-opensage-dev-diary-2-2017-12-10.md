---
title:   "OpenSAGE Dev Diary #2 (2017-12-10)"
date:    2017-12-10 11:00:00 UTC
excerpt: "What's been happening this week in OpenSAGE"
tags:    OpenSAGE
---

Following on from [last week's post](/blog/archive/2017/12/03/opensage-dev-diary-2017-12-03), here is what's been happening this week in OpenSAGE.

## Progress this week

This week, I have:

* Implemented a new build system for shaders. Previously, shaders were compiled (at build time) using the [Microsoft.HLSL.CSharpVB MSBuild targets](https://www.nuget.org/packages/Microsoft.HLSL.CSharpVB/) from NuGet. This had a couple of limitations: first, the method this NuGet package used to find `fxc.exe` [didn't work](https://github.com/OpenSAGE/OpenSAGE/issues/8) with recent Windows Kits. Second, vertex and pixel shaders had to be in separate files, which didn't fit well with the runtime Effect system I've been building. The new shader build system uses a custom MSBuild `.targets` file and a custom C# MSBuild `Task` to compile all the `.fx` files found in the OpenSage.Game project, and embed the resulting bytecode as embedded resources. [Source code here](https://github.com/OpenSAGE/OpenSAGE/tree/master/build).

* Started implementing a GUI renderer. More details on GUI rendering below.

* Started porting the existing Data Viewer application from WPF to [Eto.Forms](https://github.com/picoe/Eto), a cross-platform UI framework that, like Xamarin Forms, uses native widgets on each platform. I plan to bring OpenSAGE to the Mac soon, and this is an important preparatory step.

## GUI rendering

In C&C Generals, almost all the UI (i.e. everything except for the 3D bits) is defined in `.wnd` files. This includes the main menu, the options screen, the multiplayer game setup screen, even the command bar at the bottom of the screen during gameplay.

 `.wnd` are plain-text files that look like this:

```
FILE_VERSION = 2;
STARTLAYOUTBLOCK
  LAYOUTINIT = "[None]";
  LAYOUTUPDATE = "[None]";
  LAYOUTSHUTDOWN = "[None]";
ENDLAYOUTBLOCK
WINDOW
  WINDOWTYPE = USER;
  SCREENRECT = UPPERLEFT: 0 0,
               BOTTOMRIGHT: 800 600,
               CREATIONRESOLUTION: 800 600;
  NAME = "TheName";
  STATUS = ENABLED+IMAGE;
  STYLE = USER;
  SYSTEMCALLBACK = "[None]";
  INPUTCALLBACK = "[None]";
  TOOLTIPCALLBACK = "[None]";
  DRAWCALLBACK = "[None]";
  FONT = NAME: "Times New Roman", SIZE: 14, BOLD: 0;
  HEADERTEMPLATE = "[None]";
  TOOLTIPDELAY = -1;
  TEXTCOLOR = ENABLED:  255 255 255 255, ENABLEDBORDER:  255 255 255 255,
              DISABLED: 255 255 255 255, DISABLEDBORDER: 255 255 255 255,
              HILITE:   255 255 255 255, HILITEBORDER:   255 255 255 255;
  ENABLEDDRAWDATA = IMAGE: SomeImageName, COLOR: 0 0 128 255, BORDERCOLOR: 254 254 254 255,
                    ...
  DISABLEDDRAWDATA = IMAGE: NoImage, COLOR: 64 64 64 255, BORDERCOLOR: 254 254 254 255,
                     ...
  HILITEDRAWDATA = IMAGE: NoImage, COLOR: 128 128 255 255, BORDERCOLOR: 254 254 254 255,
                   ...
  CHILD
     ... Child window
  WINDOW
  ENDALLCHILDREN
END
```

A `WINDOW` definition (which is really an element of a window) contains information about size, colour, background image.
There are different window types for textboxes, listboxes, etc. Windows can have child windows, so something like the main menu
has a large hierarchy of "windows".

I couldn't find any information about `.wnd` files by Googling, so I had to figure it out the hard way. For example, you can see above that there are references to images (`ENABLEDDRAWDATA = IMAGE: SomeImageName`). These don't correspond directly to images on disk. Instead, there is a folder of `.ini` files called MappedImages, and in there you'll find entries like this:

```
MappedImage SomeImageName
  Texture = TheActualTexture.tga
  TextureWidth = 1024
  TextureHeight = 256
  Coords = Left:0 Top:64 Right:800 Bottom:255
  Status = NONE
End
```

These `MappedImage` entries contain the actual texture, as well as coordinates within that texture. Most GUI textures contain images for lots of GUI elements, so these coordinates provide the specific rectangle to use for a given GUI element.

I've implemented a `.wnd` parser, and started on implementing a GUI renderer. Here's where I've got to this week: the basic bits are in place, but I'm not yet rendering text, and there's no mouse or keyboard input yet.

{% responsive_image path: assets/posts/opensage-2017-12-10-gui.png alt: "OpenSAGE GUI - C&C Generals Main Menu" %}

For reference, here is what the same screen looks like in the original game:

{% responsive_image path: assets/posts/opensage-2017-12-10-gui-original.png alt: "C&C Generals Main Menu" %}

Still plenty of work to do, but it's a start.

## Next week

Next, I'm planning to finish porting the Data Viewer application to Eto.Forms, and then keep going with GUI rendering.

See you next time!