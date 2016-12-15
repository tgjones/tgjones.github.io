---
title:   "My default stack"
date:    2010-11-05 08:10:00 UTC
---

Before you write this off as one of those ego posts that... no, it's pretty much an ego post. I offer it in the chance that may be useful to somebody somewhere.

This is a list of the software / libraries I use for new projects.

## The basics

* Visual Studio 2010
* ReSharper 5.1
* .NET 4.0
* ASP.NET MVC 2.0 - I use a number of open source libraries which haven't yet updated to MVC 3.0 Beta
* [Reflector](http://www.red-gate.com/products/reflector/) - is it possible to survive without this?

## Database

* [MongoDB](http://www.mongodb.org/) - I'm a recent convert from SQL Server. The so-called NoSQL debate goes on, but for most web projects MongoDB makes a lot of sense.
* [NoRM](http://normproject.org/) - .NET driver for MongoDB

## ASP.NET

* As mentioned above, ASP.NET MVC 2.0
* [Spark](http://www.sparkviewengine.com) view engine
* [dotless](https://github.com/dotless/dotless) - I'll generally use this if I'm writing the CSS myself
* [Microsoft Ajax Minifier](http://ajaxmin.codeplex.com) - I use the MSBuild task at compile time (by the way, was there ever a worse name? It minifies JS and CSS, and really has nothing to do with AJAX)

## Code

* Dependency injection: Managed Extensibility Framework (MEF). I used to use [Ninject](http://ninject.org), and still love it, but in most cases MEF is enough, and it's built-in to .NET 4.0.
* [DynamicImage](https://github.com/sitdap/dynamic-image) - A not-well-known (so far) but excellent library from [Sound in Theory](http://www.sitdap.com) which takes care of all image manipulation.

## Testing

* [NUnit](http://www.nunit.org/) - Don't really need to say much about this one.
* [dotTrace](http://www.jetbrains.com/profiler/index.html)?topDT
* I don't use a coverage tool yet, but I'm a big fan of JetBrains, so I'll probably check out [dotCover](http://www.jetbrains.com/dotcover/) soon.

## Deployment

* MSBuild script to build solution, run NUnit tests, and call msdeploy to sync with live (or staging, depending on the project) IIS website
* TeamCity running on build server to detect changes in source code repository and run MSBuild script