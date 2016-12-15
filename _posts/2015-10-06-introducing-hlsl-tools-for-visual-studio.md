---
title:   "Introducing HLSL Tools for Visual Studio"
date:    2015-10-06 08:03:00 UTC
---

## TL/DR

HLSL Tools is a Visual Studio extension - for VS2013 and VS2015, Community Edition upwards - that provides enhanced support for editing High Level Shading Language (HLSL) files.

* You can install HLSL Tools from within Visual Studio (`Tools > Extensions and Updates`, search for "HLSL Tools")
* or download it from the [Visual Studio Gallery](https://visualstudiogallery.msdn.microsoft.com/75ddd3be-6eda-4433-a850-458b51186658)
* and [here's the GitHub repo](https://github.com/tgjones/HlslTools)

## What?

![HLSL Tools screenshot](/assets/posts/hlsl-tools.png)

Visual Studio, since VS2013, has included limited language support for HLSL. This support is built on top of the C++ editor (you can tell that in a couple of ways; changing C++ options affects HLSL, and sometimes the automatic formatting after you type a `;` or `}` is a bit off, presumably because the C++ automatic formatter doesn't recognise all HLSL syntax constructs). It's certainly better than not having it at all, but we Visual Studio users are used to a bit more. So HLSL Tools adds these features:

* **Navigation bar** - shows namespaces (yes, HLSL has them, who knew!), classes, structs, interfaces, constant buffers, global variables, functions.
  ![Navigation bar demo](https://github.com/tgjones/HlslTools/raw/master/art/navigation-bar.gif)

* **Navigate to** - activate Navigate to by typing `Ctrl+,`, and start typing a search term.
  ![Navigate to demo](https://github.com/tgjones/HlslTools/raw/master/art/navigate-to.gif)

* **Live syntax errors**
  ![Live syntax errors demo](https://github.com/tgjones/HlslTools/raw/master/art/live-errors.gif)

* **Go to definition** - currently, F12 / go to definition is only supported for preprocessor macros. More complete support is [on the roadmap](https://github.com/tgjones/HlslTools/blob/master/CHANGELOG.md).
  ![Go to definition demo](https://github.com/tgjones/HlslTools/raw/master/art/go-to-definition.gif)

* **Quick info** - again, quick info is currently only supported for preprocessor macros - more complete support is [on the roadmap](https://github.com/tgjones/HlslTools/blob/master/CHANGELOG.md).
  ![Quick info demo](https://github.com/tgjones/HlslTools/raw/master/art/quick-info.gif)

* **Preprocessor support** - the parser in HLSL Tools understands preprocessor directives, and can gray out excluded code. You can control that using the predefined `__INTELLISENSE__` macro (same as in the VS C++ editor).
  ![Preprocessor support demo](https://github.com/tgjones/HlslTools/raw/master/art/intellisense-macro.gif)

* **Options** - HLSL Tools gives HLSL its own set of options, so you can adjust things separately from C++.
  ![Options demo](https://github.com/tgjones/HlslTools/raw/master/art/options.gif)

## Why?

I've always wanted to have a go at putting together a Visual Studio language service. They seemed, if I'm honest, impossibly difficult, which is usually a reliable indicator of something that will make for a fun challenge (of course, very few types of software are actually that difficult, if you break them down into small enough pieces, and that held true here). HLSL is perhaps (?) the most popular language edited in Visual Studio that *doesn't* have its own language service. I'm familiar with HLSL from working with XNA and SharpDX, so I thought I'd have a go. I started tinkering back in April, and - [insert montage here](https://www.youtube.com/watch?v=pFrMLRQIT_k) - the result is HLSL Tools. (It might have been quicker, but it turns out having a baby, while an unqualifiedly wonderful experience, severely reduces one's hobby coding time.)

## How?

For those who are interested in such things, I wanted to call out a few of the interesting problems I encountered while developing HLSL Tools.

Almost every feature in HLSL Tools is built on top of the syntax tree produced by an HLSL parser. I initially [used ANTLR to generate a lexer and parser](https://gist.github.com/tgjones/d8df1d9a8e695b7d25ce), and this worked quite well. But I found two problems: performance, and error recovery. Every time you type a character, HLSL Tools queues up a background job to reparse the code (although to be clear, very few of those jobs will actually get executed, because it tries to be intelligent about discarding or cancelling obsolete jobs.) Parser performance makes a noticeable difference to editor responsiveness. Obviously performance is relative - and in this case, it's relative to a handwritten parser that I initially wrote as a test. I found that my handwritten lexer and parser performed quicker than the ANTLR-generated one, in almost all cases. The second aspect was error recovery. Error recovery is what happens when the parser encounters invalid code - which, in an editor, is almost all of the time. To have good error recovery means to discard as little code as possible before you can resume parsing the next valid bit of code. By using a handwritten parser, I could customise error recovery much more than I could with an ANTLR-generated parser.

Getting performance to an acceptable level was one of the big challenges in writing HLSL Tools. I tested performance using BC7Encode.hlsl, a shader from the old DirectX SDK that is 65KB in size and has 1685 lines of code. Hopefully that's at the upper end of what most people work with - so if that one works well, most other shaders should too. By comparison, the popular fxaa.hlsl is 33KB and 792 lines. The system I've ended up with, as I mentioned, is that any text changes in the editor result in a job being queued up on a background thread. Only one parse job is active at a time. If the text in the editor changes while a parse job is active, that job is cancelled, because we are only going to discard its results anyway. The code responsible for this lives in [BackgroundParser.cs](https://github.com/tgjones/HlslTools/blob/master/src/HlslTools.VisualStudio/Parsing/BackgroundParser.cs). Assuming the parse completes without being cancelled, the next step is to send the resulting syntax tree to whatever needs it. There's a priority system, so that, for example, syntax highlighting happens first (since that's the most jarring when it takes too long to update). Then the navigation bar is updated, brace matching and outlining is done, and finally syntax error squiggles and tasks in the error list are refreshed. As much work as possible is done on a background thread; some things can only be done on the UI thread, but that is kept to a minimum.

Like Roslyn, the HLSL parser in HLSL Tools creates a full fidelity syntax tree - this means that the syntax tree contains every piece of information contained in the source code, including whitespace and preprocessor directives. Preprocessor directives are evaluated as they are encountered, which enables HLSL Tools to do gray out unused code (HLSL files are often passed preprocessor defines during compilation, but you can customise what the parser "sees" using the predefined `__INTELLISENSE__` macro). And because all the information about both the original preprocessor tokens, and (in the case of macros) expanded tokens, are stored in the syntax tree, HLSL Tools supports F12 go to definition and quick info for preprocessor macros. Getting preprocessor macro expansion working correctly was one of the hardest parts of writing HLSL Tools. In C, C++, and HLSL, preprocessor macro references can appear basically anywhere, including within other macros, and so keeping track of the original macro reference, as well as the expanded macro tokens, and using the expanded tokens in the syntax tree, is fiddly and not fun code to write.

Automatic formatting is a feature that, as users of Visual Studio, we take for granted, but it's non-trivial to implement. I based my implementation on the [automatic formatter in Node.js Tools for Visual Studio](https://github.com/Microsoft/nodejstools/blob/master/Nodejs/Product/Analysis/Formatting/FormattingVisitor.cs). It still required a lot of tweaking to get it right, and it's the part of HLSL Tools that makes me most nervous - in theory, it's only supposed to remove or add whitespace, but there's always the potential for bugs, despite a reasonable set of tests around it. Automatic formatting is the main reason that the initial release of HLSL Tools is version 0.9. Once I'm satisfied that it's functioning correctly, I'll release 1.0.

Fortunately, people much smarter than I have solved a number of these problems before. Adapting, or studying and then rewriting, other people's code is much easier than writing code from scratch. I've mentioned the projects I used as references in the [acknowledgements in the readme](https://github.com/tgjones/HlslTools#acknowledgements).

## What's next?

I believe that, even in this initial release, HLSL Tools includes enough features, on top of what's in plain Visual Studio, to make it useful. But this is just the start. I expect most people's first question will be: does it include IntelliSense? The answer is: not yet. IntelliSense is the aggregate name for a number of different features, including quick info, parameter completion, statement completion, etc. The groundwork is there to support these features, but much more work remains to be done. Keep an eye on the [roadmap](https://github.com/tgjones/HlslTools#acknowledgements) if you want to track progress.

Lastly, and perhaps most importantly - if you find a bug, please [report it](https://github.com/tgjones/HlslTools/issues), including as much detail as possible. Thank you!