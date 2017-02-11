---
title:   "Try HLSL online"
date:    2017-02-11 10:00:00 UTC
excerpt: "Introducing a new website that lets you try out the old and new HLSL compilers online."
---

While developing [HLSL Tools for Visual Studio](https://github.com/tgjones/HlslTools),
I often want to check whether various bits of HLSL are valid using the official compiler.
I could create and compile an HLSL file on my computer, but that sounds like a lot of work,
so I made a website to make my life easier. Here it is:

* [Try HLSL](https://tryhlsl.azurewebsites.net)

![Screenshot of TryHLSL website](/assets/posts/try-hlsl-online.png)

You can choose to compile your HLSL code using either of Microsoft's official HLSL compilers:

* The old fxc.exe compiler that has been around for years
* [DirectXShaderCompiler](https://github.com/Microsoft/DirectXShaderCompiler), the shiny new open source compiler

You can select the shader profile and specify the shader entry point (i.e. `PSMain`, etc.).
I plan to add the ability to choose other compiler options, but that's not there yet.

At the moment, the only output option is to view the shader disassembly. I plan to add the
option to view the abstract syntax tree (AST) from the new compiler. I'm also interested in
supporting 3rd party shader processors, such as AMD's `CodeXLAnalyzer.exe`. 
Joshua Barczak's [Pyramid](https://github.com/jbarczak/Pyramid) is a really nice implementation of this sort of
multiple-input/multiple-output type of shader compiler tool.

## Implementation notes

I initially wanted to use the [Monaco text editor](https://microsoft.github.io/monaco-editor/), 
but I couldn't get it to fit properly within a flexbox layout. So I ended up using 
[CodeMirror](https://codemirror.net). To make the input and output
look nicer, I wrote a couple of CodeMirror grammars that might be interesting to others:

* [HLSL mode for CodeMirror](https://github.com/tgjones/TryHlsl/blob/master/src/OnlineHlslCompiler/Scripts/codemirror-mode-hlsl.js)
* [LLVM IR mode for CodeMirror](https://github.com/tgjones/TryHlsl/blob/master/src/OnlineHlslCompiler/Scripts/codemirror-mode-llvm-ir.js)