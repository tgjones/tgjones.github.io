---
title:   "Gemini - a.k.a. \"Make your own Visual Studio\""
date:    2013-06-21 16:08:00 UTC
excerpt: "I've published v0.3.0 of Gemini, a WPF framework designed for building IDE-like applications."
---

Wow, it's June and this is my first post of 2013! I probably should do this more often.

I just wanted to mention that I've published v0.3.0 of Gemini, a framework I've written for WPF that allows you to create IDE-like applications fairly easily.

![](/assets/520e19aef51f2777790000c5/standard/gemini-everything.png)

* [Source code](https://github.com/tgjones/gemini)
* [Documentation](http://documentup.com/tgjones/gemini)

Search for [Gemini](http://nuget.org/packages?q=Gemini) on NuGet to find all the Gemini-related packages. Note that the package name for Gemini itself is GeminiWpf, to differentiate it from another NuGet package with the same name.

Gemini has six modules built-in:

* Shell
* MainMenu
* StatusBar
* ToolBars
* Toolbox
* UndoRedo

There are several optional modules, each of which has its own NuGet package:

* CodeCompiler
* CodeEditor
* ErrorList
* GraphEditor
* Inspector
* Inspector.Xna
* Output
* PropertyGrid
* Xna

All of these are described in more detail in the [documentation](http://documentup.com/tgjones/gemini).

Please let me know if you build anything cool with this! And if you find any bugs or want to suggest new features, please let me know in the [issue tracker](https://github.com/tgjones/gemini/issues).