---
title:   "Cross-platform C# and F# in 2015"
date:    2015-09-19 15:44:00 UTC
---

C# and ([lest we forget](https://github.com/Microsoft/visualfsharp/issues/544)) F# are great choices for cross-platform development. Long gone are the days when .NET meant Windows-only - and in reality those days hardly existed at all, thanks to [Mono](http://www.mono-project.com)'s early appearance on the scene.

Given how many platforms you can now target with .NET languages, I want to try to enumerate *some* of them. This won't be an exhaustive list, but hopefully it's a good representative sample of the mainstream options. (And sorry, VB.NET fans, for not including your language in the title of this blog post. VB.NET just isn't as well supported as C# or F# when it comes to other platforms.)

If you spot something missing, leave a comment and I'll update the post.

To be clear: I have no opinion about whether you *should* use C# or F# for every program on every platform. That depends on many, many factors. This is simply about what is possible. (And just for the record, you probably shouldn't...)

## Windows

* [.NET Framework](https://en.wikipedia.org/wiki/.NET_Framework) - Every Windows release since Windows 98 and Windows NT 4.0 has supported some version of .NET. This is the "normal" way of running .NET apps - your C#, F#, or VB.NET code is compiled to MSIL, which is then executed on the Common Language Runtime (CLR). Historically, there have been multiple versions of the .NET Framework, one for Windows Desktop, one for Windows Store, one for Windows Phone, and so on.

* [.NET Core](http://blogs.msdn.com/b/dotnet/archive/2014/12/04/introducing-net-core.aspx) - this is a new implementation of .NET (including a new runtime, [CoreCLR](https://github.com/dotnet/coreclr)). Eventually, .NET Core is intended to become the [standard](https://xkcd.com/927/) version of .NET. .NET Core will run on Windows, OS X, and Linux.

* [.NET Native](http://blogs.msdn.com/b/dotnet/archive/2014/04/02/announcing-net-native-preview.aspx) - another compilation target for .NET applications, which, as the name suggests, does static compilation of .NET. **Update**: as noted by Onur in the comments, .NET Native is currently only supported for C# and VB.NET, which means you can only use those languages to write Windows 10 UWPs.

## OS X

* Mono does, and .NET Core will, allow you to run .NET programs on OS X. For example, [ASP.NET 5 will run on either Mono or .NET Core on OS X](http://docs.asp.net/en/latest/getting-started/installing-on-mac.html#install-the-net-execution-environment-dnx).

* [Xamarin.Mac](https://xamarin.com/platform#desktop) is a set of bindings to the APIs required for OS X GUI application development. Unlike its predecessor, [MonoMac](http://www.mono-project.com/docs/tools+libraries/libraries/monomac/), it also allows you to bundle the Mono framework, so you can publish apps to the Mac App Store.

## Linux

* [Mono](http://www.mono-project.com/) runs your .NET applications on Linux. In the future, .NET Core will do this too. There are a number of [.NET bindings for GUI toolkits](http://www.mono-project.com/docs/gui/). Running on Linux opens up a wide world of possibilities, including running .NET apps on Raspberry Pi.

## iOS

* [Xamarin.iOS](https://xamarin.com/platform#ios) is both a set of bindings for iOS APIs, and an ahead-of-time (AOT) compiler that compiles C# and F# code directly to ARM assembly code. It is the de-facto method of getting your .NET code to run on iOS devices.

* [RemObjects C#](http://www.elementscompiler.com/elements/hydrogene/) allows you to run C# apps on iOS, I haven't used it myself, and I'm not sure how it works, but I assume it must also use some sort of AOT compiler.

## Android

* [Xamarin.Android](https://xamarin.com/platform#android) is a set of .NET bindings for Android APIs, and a [packaging of the Mono VM that runs alongside the Dalvik VM](http://tirania.org/blog/archive/2011/Apr-06.html).

* [RemObjects C#](http://www.elementscompiler.com/elements/hydrogene/) allows you to run C# apps on Android. Unlike Xamarin.Android, [it apparently compiles your C# code to Java bytecode](http://docs.elementscompiler.com/Platforms/Java/).

## JavaScript

* [JSIL](http://www.jsil.org) translates MSIL to Javascript, which means it works (in theory, at least) with any .NET language, including F#.

* C#-to-JavaScript include [DuoCode](http://duoco.de), [Bridge.NET](http://bridge.net), and [SharpKit](http://sharpkit.net).

* **Update**: Reed Copsey reminded me in the comments about two F#-to-JavaScript offerings: [FunScript](http://funscript.info/), which is "F# to Javascript with type providers", and [WebSharper](http://websharper.com/).

## WebAssembly

* [ilwasm](https://github.com/WebAssembly/ilwasm) is early in development, but it's one to keep an eye on. It promises to compile MSIL to [WebAssembly](https://brendaneich.com/2015/06/from-asm-js-to-webassembly/).

## PlayStation 4

* [Mono runs on the PS4](http://tirania.org/blog/archive/2014/Apr-14.html), and [MonoGame takes full advantage of that](http://www.monogame.net/2014/03/23/monogame-for-playstation-4/).

## Honourable mentions

* [Unity3D](https://unity3d.com) differs from the others in this list, because it's not a platform. But it deserves a mention because it enables C# game scripts to be used on [many platforms](https://unity3d.com/unity/multiplatform). Historically, this C# support has been based on Mono. Unity are now developing [IL2CPP](http://blogs.unity3d.com/2014/05/20/the-future-of-scripting-in-unity/), which looks like an interesting new direction.
* [.NET Micro Framework](http://www.netmf.com/) allows the use of a small version of .NET in small devices, including (in the past) Microsoft's SPOT watches.

## Legacy

* Xbox 360 - [XNA](https://en.wikipedia.org/wiki/Microsoft_XNA) allowed games, written in C#, to run on Xbox 360 consoles (and Zunes - anybody remember those?). XNA was [sadly discontinued](http://www.polygon.com/2013/1/31/3939230/microsoft-has-no-plans-for-future-versions-of-xna-software), but [MonoGame has picked up the mantle](http://darkgenesis.zenithmoon.com/xna-is-no-more-as-the-phoenix-rises-from-the-ashes/), and [supports many more platforms than XNA ever did](http://www.monogame.net/about/).
* Silverlight - a technically magnificent product, but one that perhaps arrived a few years too late. You could write Silverlight apps in .NET languages, and have those apps run in (some) browsers.

## Future

More platforms and compilation targets are on the way:

* [LLILC](https://www.dotnetfoundation.org/blog/announcing-llilc-llvm-for-dotnet) is a new LLVM-based compiler for .NET. [LLILC](https://github.com/dotnet/llilc) "includes a set of cross-platform .NET code generation tools that enables compilation of MSIL byte code to LLVM supported platforms." This is a very promising project, and definitely one to watch.
* watchOS 2 and tvOS - Xamarin have already announced upcoming support for both [watchOS 2](https://blog.xamarin.com/watchos-2-preview-and-updates/) and [tvOS](https://blog.xamarin.com/now-playing-ios9-up-next-tvos/).
* Xbox One - ironically, while it is possible to write PS4 games using C# today, you can't do that on Microsoft's Xbox One. It has been confirmed that Windows 10 apps, including .NET apps, will be able to run on the Xbox One, after a future system update. But it's still not clear .NET will be allowed in the [game-specific partition](http://wccftech.com/xbox-one-architecture-explained-runs-windows-8-virtually-indistinguishable/).

C# has come a long way since its introduction in 2000. F#, likewise, has been getting a lot of cross-platform love since it appeared in 2005. I'm excited to see what the next decade holds.