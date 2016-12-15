---
title:   "Introducing DotLiquid: the secure, open-source template engine for .NET"
date:    2010-10-25 23:11:00 UTC
---

Want to skip straight to the goods?

* [Try DotLiquid online](http://dotliquidmarkup.org/try-online)
* [Download DotLiquid](http://github.com/formosatek/dotliquid/downloads)

## The Problem

One of the questions that comes up fairly often for the average developer is: "How can I transfer *this* data into *that* output?" The data often takes the form of an object, and the output is often HTML or plain text. There are, of course, [many](http://whereslou.com/2009/01/27/late-bound-eval-email-templates-revisited) [ways](http://code.google.com/p/csharptemplates/) [of](http://www.stringtemplate.org/) [skinning](http://csharp-source.net/open-source/template-engines) [the cat](http://visualstudiomagazine.com/articles/2009/05/01/visual-studios-t4-code-generation.aspx)[^1].

These existing .NET template engines are very suitable for one specific use-case: when the developer, or another trusted party, is responsible for authoring the templates. However, to the best of my knowledge, there have not been any full-featured[^2] template engines available for .NET which are suitable for the other use-case: when *you want customers to edit their own templates*. You may want this because you host the customer's website on your server, or simply because .NET is too complex to give to a non-techie.

## Enter DotLiquid

This is the thinking behind why I wrote [DotLiquid](http://dotliquidmarkup.org/), a C# port of the [Liquid](http://www.liquidmarkup.org) template engine from Ruby. DotLiquid offers the following benefits:

* It is *secure*. It only allows access to the data and operations you explicitly allow.
* It is *proven*. The [Liquid](http://www.liquidmarkup.org/) template language has been used in major web applications such as [Shopify](http://www.shopify.com/).
* It can be *extended easily*, in part because it is open source, in part because it's [designed that way](http://github.com/formosatek/dotliquid/wiki/DotLiquid-for-Developers).
* It is completely *standalone*.

## Where can I find it?

The website is [here](http://dotliquidmarkup.org/). You can <a href="http://dotliquidmarkup.org/try-online">try it online</a>. The source code for DotLiquid is hosted on GitHub, and can be found <a href="http://github.com/formosatek/dotliquid">here</a>. This is also the place to <a href="http://github.com/formosatek/dotliquid/issues">log issues</a>. There is some <a href="http://github.com/formosatek/dotliquid/wiki">documentation</a>, although (as ever) I hope to expand on what is there. The current version of DotLiquid can be downloaded <a href="http://github.com/formosatek/dotliquid/downloads">here</a> (we're on Beta 2 at the moment, and we're just letting that brew for a while before pushing out the final release of 1.0). Once "<a href="http://nupack.codeplex.com/">the package manager formerly known as NuPack</a>" has settled down a bit, I'll be creating a package so that you can easily include DotLiquid in your projects.

If you've made it this far, chances are you might be looking for this sort of library. If you do use it in your project, I hope you find it helpful, and if you would like your project to be mentioned on the <a href="http://dotliquidmarkup.org/">DotLiquid website</a>, just <a href="http://formosatek.com/">drop me a line</a>.

## Acknowledgements

DotLiquid wouldn't exist without Liquid, and Liquid wouldn't exist without <a href="http://blog.leetsoft.com/">Tobias Lütke</a>. Thank you Tobias.

I'd also like to thank <a href="http://www.alessandropetrelli.com/">Alessandro Petrelli</a> for his many useful contributions to DotLiquid.

[^1]: I actually like cats.

[^2]: I say "full-featured" because I could find at least one [template engine which uses Django-style syntax](http://code.google.com/p/csharptemplates/), but I don't think it offers important constructs such as loops and conditionals.