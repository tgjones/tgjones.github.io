---
title:   "HLSL Tools for Visual Studio v1.0 released"
date:    2016-04-25 10:41:00 UTC
---

I'm very happy to announce the release of version 1.0 of HLSL Tools for Visual Studio:

* You can install HLSL Tools from within Visual Studio (`Tools > Extensions and Updates`, search for “HLSL Tools”)
* or download it from the [Visual Studio Gallery](https://visualstudiogallery.msdn.microsoft.com/75ddd3be-6eda-4433-a850-458b51186658)
* or [grab the source code from the GitHub repo](https://github.com/tgjones/HlslTools) and build it yourself.

If you edit HLSL shaders in Visual Studio, then this extension is for you. Please go and download it! Did I mention it includes IntelliSense for HLSL? And then come back here to find out what's new in version 1.0.

## What is HLSL Tools?

For those who missed my initial [announcement blog post](http://timjones.tw/blog/archive/2015/10/06/introducing-hlsl-tools-for-visual-studio), here's a brief recap. Visual Studio, since VS2013, has shipped with basic support for editing HLSL shaders. This basic support includes syntax highlighting, mostly-correct automatic formatting, brace completion, brace matching, and outlining.

HLSL Tools for Visual Studio is a free (and open source) Visual Studio extension that improves upon this. Since its initial release, HLSL Tools has included these features:

* Navigation bar
* Navigate to (`Ctrl+,`)
* Live syntax errors
* Preprocessor support (code excluded by the preprocessor is greyed out)
* HLSL-specific text editor options

## What's new?

So what's new in v1.0? Quite a lot, as it turns out: most of the new features can be TL/DR'd as "IntelliSense", but there are other improvements too.

### Statement completion

Just start typing, and HLSL Tools will show you a list of the available symbols (variables, functions, etc.)
at that location. You can manually trigger this with the usual shortcuts: `Ctrl+J`, `Ctrl+Space`, etc.

![Statement completion demo](https://github.com/tgjones/HlslTools/raw/master/art/statement-completion.gif)

### Signature help

Signature help (a.k.a. parameter info) shows you all the overloads for a function call, along with information (from MSDN, for intrinsic functions)
about the function, its parameters, and return types. Typing an open parenthesis will trigger statement
completion, as will the standard `Ctrl+Shift+Space` shortcut. Signature help is available for all HLSL functions and methods,
including the older `tex2D`-style texture sampling functions, and the newer `Texture2D.Sample`-style methods. Signature help also works for your own functions.

![Signature help demo](https://github.com/tgjones/HlslTools/raw/master/art/signature-help.gif)

### Reference highlighting

Placing the cursor within a symbol (local variable, function name, etc.) will cause all references to
that symbol to be highlighted. Navigate between references using `Ctrl+Shift+Up` and `Ctrl+Shift+Down`.

![Reference highlighting demo](https://github.com/tgjones/HlslTools/raw/master/art/reference-highlighting.gif)

### Live semantic errors

HLSL Tools shows you syntax and semantic errors immediately. No need to wait till compilation!
Errors are shown as squigglies and in the error list. HLSL Tools will also show you implicit conversion warnings. Instead of the generic "implicit vector truncation" warning that the HLSL compiler gives you, HLSL Tools tells you what the source and destination types are.

![Live errors demo](https://github.com/tgjones/HlslTools/raw/master/art/live-errors.gif)

### Go to definition

Press F12 to go to a symbol definition. Go to definition works for variables, fields, functions, classes,
macros, and more.

![Go to definition demo](https://github.com/tgjones/HlslTools/raw/master/art/go-to-definition.gif)

### Quick info

Hover over almost anything (variable, field, function call, macro, semantic, type, etc.) to see a Quick Info tooltip.

![Quick info demo](https://github.com/tgjones/HlslTools/raw/master/art/quick-info.gif)

## What's next?

There are more features on the [roadmap](https://github.com/tgjones/HlslTools/blob/master/CHANGELOG.md). If HLSL Tools is useful to you, I'd leave to hear about it! And if you have any feature suggestions, or find any bugs, please [create an issue on the GitHub repository](https://github.com/tgjones/HlslTools/issues).

You can also [follow me on Twitter](https://twitter.com/_tim_jones_), where I tweet about HLSL Tools using the hashtag [#hlsltools](https://twitter.com/hashtag/hlsltools).