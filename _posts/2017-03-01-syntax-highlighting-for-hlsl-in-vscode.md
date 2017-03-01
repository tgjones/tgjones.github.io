---
title:   "Syntax highlighting for HLSL in Visual Studio Code"
date:    2017-03-01 22:25:00 UTC
excerpt: "My pull request to add syntax highlighting to HLSL files in VSCode was merged."
---

I've started in recent weeks to use [VSCode](https://code.visualstudio.com) much more frequently.
It's a great lightweight editor, but it still has enough extensions to make you feel productive at
a wide range of tasks.

VSCode has had built-in syntax highlighting for ShaderLab `.shader` files for some time now. But while this built-in support was definitely better than not having syntax highlighting at all, it left quite a bit to be desired. And VSCode has never had built-in syntax highlighting for HLSL shaders (despite HLSL being a Microsoft language!). (Yes, I do know about Stef Levesque's excellent [vscode-shader](https://marketplace.visualstudio.com/items?itemName=slevesque.shader) extension, which includes HLSL, GLSL, and Cg syntax highlighting; but I'm specifically talking about what's available in VSCode core.)

I wanted to both improve ShaderLab syntax highlighting, and add HLSL syntax highlighting, to VSCode, and I'm [happy to say](https://code.visualstudio.com/updates/v1_10#_thank-you) that the [February 2017 release of VSCode](https://code.visualstudio.com/updates/v1_10) includes my [pull request](https://github.com/Microsoft/vscode/pull/20129) that adds exactly these features.

That's enough text, here are some pictures. First of an HLSL file in the January 2017 release of VSCode:

![](/assets/posts/vscode-hlsl-before.png)

And now in the February 2017 release:

![](/assets/posts/vscode-hlsl-after.png)

And here's a ShaderLab file in the February 2017 release:

![](/assets/posts/vscode-shaderlab-after.png)

This is the first result of my efforts to give a better experience to shader authors using VSCode. I'm prototyping a full language service for HLSL and ShaderLab, based on my existing work for [HLSL Tools](https://marketplace.visualstudio.com/items?itemName=TimGJones.HLSLToolsforVisualStudio). I'll have more to say on that once I actually have something working.

Happy coding!