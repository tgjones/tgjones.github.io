---
title:   "Overriding the meaning of ~ in ASP.NET"
date:    2007-04-05 17:35:00 UTC
---

Random techie discovery... You can override the meaning of the tilde (~) in paths in ASP.NET.

I often use the `~` so that it doesn't matter whether my web application is sitting at the root level or in a virtual directory. For example, I could include this as the href of a link tag to make ASP.NET write out the correct path of a stylesheet:

``` csharp
~/App_Assets/Css/Screen.css
```

I discovered today that you can override the meaning of that `~`, to potentially point anywhere you like, such as a subfolder. [See here](http://blogs.msdn.com/davidebb/archive/2005/11/27/497339.aspx) for details.