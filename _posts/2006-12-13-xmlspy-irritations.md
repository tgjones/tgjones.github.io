---
title:   "XMLSpy irritations"
date:    2006-12-13 12:47:00 UTC
---

For a project at work, I need to automate Altova XMLSpy using its API. I want to "flatten" an XSD, which includes other XSDs, into a single XSD that includes all the element definitions inline, without any duplications.

It looked like a simple job - Altova have documented their API pretty well, and it's quite clean and easy to use. However, I spent 3 hours yesterday wondering why, when I added or removed nodes, nothing happened.

It turns out - as helpfully documented in a rather hidden place - that you must switch to the Grid view before you can use XMLData objects to programmatically alter the document tree. So when you open a document, you must do this (assuming you're writing in C#):

``` csharp
Document lMyNewXsd = lApp.Documents.NewFile("test.xsd", "xsd");
lMyNewXsd.SwitchViewMode(SPYViewModes.spyViewGrid);
```

Grr - I do wish gotcha's like this were documented more clearly!