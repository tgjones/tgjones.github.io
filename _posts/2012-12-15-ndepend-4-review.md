---
title:   "NDepend 4 review"
date:    2012-12-15 15:47:00 UTC
excerpt: "NDepend is an analysis tool for .NET code, that provides many ways of generating and viewing code metrics. Here I review version 4."
---

[NDepend](http://www.ndepend.com) is an analysis tool for .NET code, that provides many ways of generating and viewing code metrics. Here's what the homepage says:

> NDepend is a Visual Studio tool to manage complex .NET code and achieve high Code Quality. With NDepend, software quality can be measured using Code Metrics, visualized using Graphs and Treemaps, and enforced using standard and custom Rules. 

I'll admit, when I did my first analysis session with NDepend, I was a little taken aback at the amount of detail being thrown at me. However, it didn't take long before I appreciated the sheer depth of available information. And viewed in that light, the visual interface is actually well thought through. It's a 4th generation product, and it shows.

First things first: when you download NDepend, you won’t find an installer. Personally, I like the convenience of an .msi, but it’s not a big deal. I chose to copy NDepend to Program Files.

NDepend 4 comes in two versions: standalone, and integrated with Visual Studio. Personally I prefer the standalone version – there are enough tool windows that I think it deserves its own host program.

When you first open NDepend, you'll be greeted by a familiar IDE-style interface.

![NDepend interface](/assets/posts/ndepend1.png)

Analysing a Project
-------------------

For the purposes of this review, I'll be analysing [DotLiquid](http://dotliquidmarkup.org). It's quite a small project, but probably large enough to have some issues that NDepend might uncover. You can analyse either a .csproj or a compiled DLL.

After analysis, NDepend will open an HTML report in your browser:

![HTML report](/assets/posts/ndepend5.png)

You can use Visual NDepend to browse the same information as shown in the report. The rest of this review will focus on Visual NDepend.

Code Query LINQ
---------------

One of NDepend 4's flagship features is Code Query LINQ (CQLinq). This, as the name suggests, lets you use LINQ to query your codebase. NDepend comes with many built-in queries, and you can write your own. 

The built-in queries cover many categories, including among others:

* Code Quality
* Object Oriented Design
* API Breaking Changes
* Dead Code
* Naming Conventions

As an example of what CQLinq looks like, here is one of the built-in rules that suggests avoiding namespaces with few types:

``` csharp
// <Name>Avoid namespaces with few types</Name>
warnif count > 0 from n in JustMyCode.Namespaces 
let types = n.ChildTypes.Where(t => !t.IsGeneratedByCompiler)
where 
  types.Count() < 5 
  orderby types.Count() ascending
select new { n, types } 
```

Selecting a query in the "Queries and Rules Explorer" panel will highlight (in the "Metrics" panel) the methods, types, namespaces or assemblies matched by that query. After selecting the "Quick summary of methods to refactor" query, this is what the "Metrics" panel looks like for DotLiquid:

![Metrics panel](/assets/posts/ndepend2.png)

Dependency Matrix
-----------------

The Dependency Matrix panel is more useful than it first appears. It shows you whether, and how, each part (namespace, type, method) of your project depends on other parts of your project or other libraries. For example, the `Extends` tag in DotLiquid depends on `IFileSystem`, so there is a blue 1 in that column and row to indicate that 1 member in the `Extends` type uses the `IFileSystem` type.

![Dependency Matrix](/assets/posts/ndepend3.png)

The Dependency Matrix is perhaps most useful for spotting dependencies that you didn't expect, or that shouldn't exist.

Dependency Graph
----------------

The Dependency Graph panel is more straightforward - as far as I can see, it's a graphical version of the Dependency Matrix panel, although it gives you several more display options. You can drill into each assembly, and view dependencies right down to the method level. You get a bird's-eye view of the relative size of each assembly / type / method, as well as a graphic depiction of their inter-dependencies.

![Dependency Graph](/assets/posts/ndepend4.png)

Code Diff
---------

With the DotLiquid project, we have committed to semantic versioning. NDepend can help with this – I can compare previous versions with the trunk version, and check that I haven’t accidentally broken API compatibility in a minor version bump. Here's what it looks like – note that you can filter by added code, removed code, changed code, etc. You can even edit the CQLinq query to completely customise the filter. 

![Code Diff](/assets/posts/ndepend6.png)

Build Server Integration
------------------------

NDepend can be integrated with TFS, CruiseControl, FinalBuilder and TeamCity. It can also be run from the command line.

Conclusion
----------

Almost everything in NDepend's interface is clickable, or has a tooltip. There is a huge amount of depth here. The documentation is very thorough – and, for once, it’s not full of useless MSDN-style "This gets or sets the [property name]" tidbits.

Overall, I am impressed with NDepend 4. The only downside I can see is its complexity (and possibly, depending on your situation, its price – but I'm all for software developers charging for their hard work). If you are willing to spend some time learning how it works, and if your project warrants it, then you'll be rewarded with some very deep code metrics. The tagline is "Make your .NET code Beautiful with NDepend." If its code metrics help you write code that can withstand the microscopic level of detail that NDepend exposes, then "beautiful" might just be the right word.

### Disclaimer

Patrick Smacchia contacted me about reviewing NDepend. I received a free licence for NDepend 4. Patrick did not express any expectation of a positive review.