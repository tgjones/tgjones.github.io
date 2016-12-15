---
title:   "ASP.NET HTML Control Mapping"
date:    2008-01-02 16:47:00 UTC
---

Ever wondered how ASP.NET takes an HTML tag, such as `<head>`, and maps it to an `System.Web.UI.HtmlControls.HtmlHead` object? Well, me neither, till today. But if you ever want to know, Reflector comes in handy. In the `System.Web.UI` namespace, there's an internal class called `HtmlTagNameToTypeMapper`. In here, there's a method called `GetControlType`, which looks like this:

``` csharp
Type ITagNameToTypeMapper.GetControlType(string tagName, IDictionary attributeBag)
{
  Type type;
  if (tagMap == null)
  {
      Hashtable hashtable = new Hashtable(10, StringComparer.OrdinalIgnoreCase);
      hashtable.Add("a", typeof(HtmlAnchor));
      hashtable.Add("button", typeof(HtmlButton));
      hashtable.Add("form", typeof(HtmlForm));
      hashtable.Add("head", typeof(HtmlHead));
      hashtable.Add("img", typeof(HtmlImage));
      hashtable.Add("textarea", typeof(HtmlTextArea));
      hashtable.Add("select", typeof(HtmlSelect));
      hashtable.Add("table", typeof(HtmlTable));
      hashtable.Add("tr", typeof(HtmlTableRow));
      hashtable.Add("td", typeof(HtmlTableCell));
      hashtable.Add("th", typeof(HtmlTableCell));
      tagMap = hashtable;
  }
  if (inputTypes == null)
  {
      Hashtable hashtable2 = new Hashtable(10, StringComparer.OrdinalIgnoreCase);
      hashtable2.Add("text", typeof(HtmlInputText));
      hashtable2.Add("password", typeof(HtmlInputPassword));
      hashtable2.Add("button", typeof(HtmlInputButton));
      hashtable2.Add("submit", typeof(HtmlInputSubmit));
      hashtable2.Add("reset", typeof(HtmlInputReset));
      hashtable2.Add("image", typeof(HtmlInputImage));
      hashtable2.Add("checkbox", typeof(HtmlInputCheckBox));
      hashtable2.Add("radio", typeof(HtmlInputRadioButton));
      hashtable2.Add("hidden", typeof(HtmlInputHidden));
      hashtable2.Add("file", typeof(HtmlInputFile));
      inputTypes = hashtable2;
  }
  if (StringUtil.EqualsIgnoreCase("input", tagName))
  {
      string str = (string) attributeBag["type"];
      if (str == null)
      {
          str = "text";
      }
      type = (Type) inputTypes[str];
      if (type == null)
      {
          throw new HttpException(SR.GetString("Invalidtypeforinputtag", new object[] { str }));
      }
      return type;
  }
  type = (Type) tagMap[tagName];
  if (type == null)
  {
      type = typeof(HtmlGenericControl);
  }
  return type;
}
```

Unfortunately, there's no way I can see of overriding this behaviour - for example, if you wanted to map the `<li>` element to a custom `HtmlListItemControl`, it's impossible.