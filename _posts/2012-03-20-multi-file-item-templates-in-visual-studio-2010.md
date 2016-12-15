---
title:   "Multi-file item templates in Visual Studio 2010"
date:    2012-03-20 14:01:00 UTC
excerpt: "Visual Studio 2010 comes with lots of item templates, but sometimes the built-in ones aren't enough.  Here I'll walk through creating a multi-file item template that will add a model, view, view model and controller to an ASP.NET MVC website."
---

Visual Studio 2010 comes with lots of item templates, but sometimes the built-in ones aren't enough.  Here I'll walk through creating a multi-file item template that will add a model, view, view model and  controller to an ASP.NET MVC website. It's not a great example, because MVC comes with its own scaffolding system for such tasks, but it will demonstrate the general technique.

Files
-----

First, create a new folder somewhere with the following structure.

* MyTemplate.vstemplate
* __TemplateIcon.ico
* Controllers
    * EntityController.cs
* Models
    * Entity.cs
* ViewModels
    * EntityViewModel.cs
* Views
    * Entity
        * Index.cshtml

The .vstemplate file
--------------------

The `.vstemplate` file should look like this:

``` csharp
<?xml version="1.0" encoding="utf-8"?>
<VSTemplate Version="3.0.0" Type="Item" xmlns="http://schemas.microsoft.com/developer/vstemplate/2005">
	<TemplateData>
		<Name>Example Entity Template</Name>
		<Description>Creates a model and related MVC items (controller, view model and view)</Description>
		<Icon>__TemplateIcon.ico</Icon>
		<ProjectType>CSharp</ProjectType>
		<SortOrder>10</SortOrder>
		<DefaultName>Entity</DefaultName>
	</TemplateData>
	<TemplateContent>
		<ProjectItem TargetFileName="Models\$fileinputname$.cs" ReplaceParameters="true">Models\Entity.cs</ProjectItem>
		<ProjectItem TargetFileName="Controllers\$fileinputname$Controller.cs" ReplaceParameters="true">Controllers\EntityController.cs</ProjectItem>
		<ProjectItem TargetFileName="ViewModels\$fileinputname$ViewModel.cs" ReplaceParameters="true">ViewModels\EntityViewModel.cs</ProjectItem>
		<ProjectItem TargetFileName="Views\$fileinputname$\Index.cshtml" ReplaceParameters="true">Views\Entity\Index.cshtml</ProjectItem>
	</TemplateContent>
	<WizardExtension>
		<Assembly>Microsoft.VisualStudio.Web.Application, Version=10.0.0.0, Culture=neutral, PublicKeyToken=b03f5f7f11d50a3a</Assembly>
		<FullClassName>Microsoft.VisualStudio.Web.Application.WATemplateWizard</FullClassName>
	</WizardExtension>
</VSTemplate>
```

Note the `WizardExtension` element. This is important, because it will allow you to use the `$classname$` parameter in the template files.

Templates
---------

I won't include all the template files here, but I'll include two so that you get the general idea. First, here is the model:

``` csharp
using MyCmsRulez;

namespace $rootnamespace$.Models
{
	public class $safeitemname$
	{
		
	}
}
```

And here is the controller. Note the use of `$classname$`: this will be replaced with the name that the user entered in the Add New Item dialog, as opposed to the name of the file currently being generated. As far as I can see, [this page](http://msdn.microsoft.com/en-us/library/eehb4faa.aspx) is not accurate in that `$itemname$` is *not* the name the user entered in the Add New Item dialog if it's a multi-item template.

``` csharp
using System.Web.Mvc;
using $rootnamespace$.Models;
using $rootnamespace$.ViewModels;

namespace $rootnamespace$.Controllers
{
	public class $classname$Controller : Controller
	{
		public ActionResult Index()
		{
			return View(new $classname$ViewModel(CurrentItem));
		}
	}
}
```

Installation
------------

* Zip up the files (making sure you don't have an unnecessary root folder), and name the file something like "EntityTemplate.zip".
* Copy into: `C:\Users\[username]\Documents\Visual Studio 2010\Templates\ItemTemplates\Visual C#`

A more elegant solution is to use the [Visual Studio 2010 SDK](http://www.microsoft.com/download/en/details.aspx?id=21835). The SDK comes with a "C# Item Template" project type. That allows your template files to be included in a project in your solution. When you build that project, it will create a `.vsix` package, which can be distributed. When run, the templates in the `.vsix` package will install themselves to the correct location.