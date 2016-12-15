---
title:   "Customising the AxWebBrowser"
date:    2006-08-15 15:22:00 UTC
---

I recently needed to use AxWebBrowser component to do some automation of web page form filling. I wanted to capture Javascript alert boxes, grab the text from the alert box, and return it as an error to the user of my application.

I also wanted to disable the loading of images and Flash animations. The code for this is below, in the `IDispatch_Invoke_Handler` method.

It took me a long time to track down the method of doing this, so I thought I'd put it here in case anybody else needs the same info.

First, some links I found useful:

* <a href="http://starkravingfinkle.org/blog/2004/09/">Mark Finkle's blog</a>
* <a href="http://www.codeproject.com/csharp/ExtendedWebBrowser.asp">Extended WebBrowser Control</a>
* <a href="http://support.microsoft.com/?kbid=279535">How to suppress run-time script errors as a WebBrowser control host</a>
* <a href="http://support.microsoft.com/kb/261003/">How to handle script errors as a WebBrowser control host</a>
* <a href="http://msdn.microsoft.com/workshop/browser/mshtml/overview/overview.asp">About MSHTML</a>

The following code shows how to setup the web browser such that you are alerted when Javascript popups are shown.

First, you need to define some constants and interfaces.

``` csharp
public enum ComConstants : uint
{
 DLCTL_DLIMAGES                          = 0x00000010,
 DLCTL_VIDEOS                            = 0x00000020,
 DLCTL_BGSOUNDS                          = 0x00000040,
 DLCTL_NO_SCRIPTS                        = 0x00000080,
 DLCTL_NO_JAVA                           = 0x00000100,
 DLCTL_NO_RUNACTIVEXCTLS                 = 0x00000200,
 DLCTL_NO_DLACTIVEXCTLS                  = 0x00000400,
 DLCTL_DOWNLOADONLY                      = 0x00000800,
 DLCTL_NO_FRAMEDOWNLOAD                  = 0x00001000,
 DLCTL_RESYNCHRONIZE                     = 0x00002000,
 DLCTL_PRAGMA_NO_CACHE                   = 0x00004000,
 DLCTL_NO_BEHAVIORS                      = 0x00008000,
 DLCTL_NO_METACHARSET                    = 0x00010000,
 DLCTL_URL_ENCODING_DISABLE_UTF8         = 0x00020000,
 DLCTL_URL_ENCODING_ENABLE_UTF8          = 0x00040000,
 DLCTL_NOFRAMES                          = 0x00080000,
 DLCTL_FORCEOFFLINE                      = 0x10000000,
 DLCTL_NO_CLIENTPULL                     = 0x20000000,
 DLCTL_SILENT                            = 0x40000000,
 DLCTL_OFFLINEIFNOTCONNECTED             = 0x80000000,
 DLCTL_OFFLINE                           = 0x80000000
}
```

``` csharp
public enum DialogBoxCommandId
{
 Ok = 1,
 Cancel = 2,
 Abort = 3,
 Retry = 4,
 Ignore = 5,
 Yes = 6,
 No = 7,
 TryAgain = 10,
 Continue = 11
}
```

``` csharp
using System.Runtime.InteropServices;
using mshtml;

[ComImport,
Guid("C4D244B0-D43E-11CF-893B-00AA00BDCE1A"),
InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
public interface IDocHostShowUI
{
 [PreserveSig]
 uint ShowMessage(IntPtr hwnd, 
  [MarshalAs(UnmanagedType.LPWStr)] string lpstrText, 
  [MarshalAs(UnmanagedType.LPWStr)] string lpstrCaption, 
  uint dwType, 
  [MarshalAs(UnmanagedType.LPWStr)] string lpstrHelpFile, 
  uint dwHelpContext,
  out int lpResult);
                                  
 [PreserveSig]
 uint ShowHelp(IntPtr hwnd, [MarshalAs(UnmanagedType.LPWStr)] string pszHelpFile, 
  uint uCommand, uint dwData, 
  tagPOINT ptMouse, 
  [MarshalAs(UnmanagedType.IDispatch)] object pDispatchObjectHit);                       
}
```

``` csharp
using System.Runtime.InteropServices;
using System.Windows.Forms;
using mshtml;

public enum DOCHOSTUITYPE
{
 DOCHOSTUITYPE_BROWSE = 0,
 DOCHOSTUITYPE_AUTHOR = 1
}

public enum DOCHOSTUIDBLCLK
{
 DOCHOSTUIDBLCLK_DEFAULT = 0,
 DOCHOSTUIDBLCLK_SHOWPROPERTIES = 1,
 DOCHOSTUIDBLCLK_SHOWCODE = 2
}

public enum DOCHOSTUIFLAG
{
 DOCHOSTUIFLAG_DIALOG = 0x00000001,
 DOCHOSTUIFLAG_DISABLE_HELP_MENU = 0x00000002,
 DOCHOSTUIFLAG_NO3DBORDER = 0x00000004,
 DOCHOSTUIFLAG_SCROLL_NO = 0x00000008,
 DOCHOSTUIFLAG_DISABLE_SCRIPT_INACTIVE = 0x00000010,
 DOCHOSTUIFLAG_OPENNEWWIN = 0x00000020,
 DOCHOSTUIFLAG_DISABLE_OFFSCREEN = 0x00000040,
 DOCHOSTUIFLAG_FLAT_SCROLLBAR = 0x00000080,
 DOCHOSTUIFLAG_DIV_BLOCKDEFAULT = 0x00000100,
 DOCHOSTUIFLAG_ACTIVATE_CLIENTHIT_ONLY = 0x00000200,
 DOCHOSTUIFLAG_OVERRIDEBEHAVIORFACTORY = 0x00000400,
 DOCHOSTUIFLAG_CODEPAGELINKEDFONTS = 0x00000800,
 DOCHOSTUIFLAG_URL_ENCODING_DISABLE_UTF8 = 0x00001000,
 DOCHOSTUIFLAG_URL_ENCODING_ENABLE_UTF8 = 0x00002000,
 DOCHOSTUIFLAG_ENABLE_FORMS_AUTOCOMPLETE = 0x00004000,
 DOCHOSTUIFLAG_ENABLE_INPLACE_NAVIGATION = 0x00010000,
 DOCHOSTUIFLAG_IME_ENABLE_RECONVERSION = 0x00020000,
 DOCHOSTUIFLAG_THEME = 0x00040000,
 DOCHOSTUIFLAG_NOTHEME = 0x00080000,
 DOCHOSTUIFLAG_NOPICS = 0x00100000,
 DOCHOSTUIFLAG_NO3DOUTERBORDER = 0x00200000,
 DOCHOSTUIFLAG_DELEGATESIDOFDISPATCH = 0x00400000
}

[StructLayout(LayoutKind.Sequential)]
public struct DOCHOSTUIINFO
{
 public uint cbSize;
 public uint dwFlags;
 public uint dwDoubleClick;
 [MarshalAs(UnmanagedType.BStr)] public string pchHostCss;
 [MarshalAs(UnmanagedType.BStr)] public string pchHostNS;
}

[StructLayout(LayoutKind.Sequential)]
public struct tagMSG
{
 public IntPtr hwnd;
 public uint message;
 public uint wParam;
 public int lParam;
 public uint time;
 public tagPOINT pt;
}

[ComImport(),
InterfaceType(ComInterfaceType.InterfaceIsIUnknown),
GuidAttribute("bd3f23c0-d43e-11cf-893b-00aa00bdce1a")]
public interface IDocHostUIHandler
{
 [PreserveSig]
 uint ShowContextMenu(uint dwID, ref tagPOINT ppt,
  [MarshalAs(UnmanagedType.IUnknown)]  object pcmdtReserved,
  [MarshalAs(UnmanagedType.IDispatch)] object pdispReserved);

 void GetHostInfo(ref DOCHOSTUIINFO pInfo);
 void ShowUI(uint dwID, ref object pActiveObject, ref object pCommandTarget, ref object pFrame, ref object pDoc);
 void HideUI();
 void UpdateUI();
 void EnableModeless(int fEnable);
 void OnDocWindowActivate(int fActivate);
 void OnFrameWindowActivate(int fActivate);
 void ResizeBorder(ref tagRECT prcBorder, int pUIWindow, int fRameWindow);

 [PreserveSig]
 uint TranslateAccelerator(ref tagMSG lpMsg, ref Guid pguidCmdGroup, uint nCmdID);

 void GetOptionKeyPath([MarshalAs(UnmanagedType.BStr)] ref string pchKey, uint dw);
 object GetDropTarget(ref object pDropTarget);
 object GetExternal();

 [PreserveSig]
 uint TranslateUrl(uint dwTranslate, 
  [MarshalAs(UnmanagedType.BStr)] string pchURLIn,
  [MarshalAs(UnmanagedType.BStr)] ref string ppchURLOut);

 IDataObject FilterDataObject(IDataObject pDO);
}

[ComImport(),
GuidAttribute("3050f6d0-98b5-11cf-bb82-00aa00bdce0b")]
public interface IDocHostUIHandler2: IDocHostUIHandler
{
 void GetOverrideKeyPath([MarshalAs(UnmanagedType.BStr)] ref string pchKey, uint dw);
}
```

``` csharp
using System.Runtime.InteropServices;

[ComImport,
Guid("00000118-0000-0000-C000-000000000046"),
InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
public interface IOleClientSite
{
 void SaveObject();
 void GetMoniker(uint dwAssign, uint dwWhichMoniker, ref object ppmk);
 void GetContainer(ref object ppContainer);
 void ShowObject();
 void OnShowWindow(bool fShow);
 void RequestNewObjectLayout();
}
```

``` csharp
using System.Runtime.InteropServices;

[ComImport,
Guid("b722bcc7-4e68-101b-a2bc-00aa00404770"),
InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
public interface IOleDocumentSite
{
 void ActivateMe(ref object pViewToActivate);
}
```

``` csharp
using System.Runtime.InteropServices;
using System.Windows.Forms;

[ComImport,
Guid("00000112-0000-0000-C000-000000000046"),
InterfaceType(ComInterfaceType.InterfaceIsIUnknown)]
public interface IOleObject
{
 void SetClientSite(IOleClientSite pClientSite);
 void GetClientSite(ref IOleClientSite ppClientSite);
 void SetHostNames(object szContainerApp, object szContainerObj);
 void Close(uint dwSaveOption);
 void SetMoniker(uint dwWhichMoniker, object pmk);
 void GetMoniker(uint dwAssign, uint dwWhichMoniker, object ppmk);
 void InitFromData(IDataObject pDataObject, bool fCreation, uint dwReserved);
 void GetClipboardData(uint dwReserved, ref IDataObject ppDataObject);
 void DoVerb(uint iVerb, uint lpmsg, object pActiveSite, uint lindex, uint hwndParent, uint lprcPosRect);
 void EnumVerbs(ref object ppEnumOleVerb);
 void Update();
 void IsUpToDate();
 void GetUserClassID(uint pClsid);
 void GetUserType(uint dwFormOfType, uint pszUserType);
 void SetExtent(uint dwDrawAspect, uint psizel);
 void GetExtent(uint dwDrawAspect, uint psizel);
 void Advise(object pAdvSink, uint pdwConnection);
 void Unadvise(uint dwConnection);
 void EnumAdvise(ref object ppenumAdvise);
 void GetMiscStatus(uint dwAspect,uint pdwStatus);
 void SetColorScheme(object pLogpal);
}
```

Now, you can define the object that will host the web browser, using the interfaces you have just declared.

``` csharp
using System.Runtime.InteropServices;
using System.Windows.Forms;
using mshtml;

public class WebBrowser:
 IOleClientSite,    // so MsHtml uses IDocHostUIHandler
 IDocHostUIHandler,
 IOleDocumentSite,  // so MsHtml uses IDocHostShowUI
 IDocHostShowUI
{
 #region IOleClientSite methods

 void IOleClientSite.SaveObject() {}
 void IOleClientSite.GetMoniker(uint dwAssign, uint dwWhichMoniker, ref object ppmk) {}
 void IOleClientSite.GetContainer(ref object ppContainer) {ppContainer = this;}
 void IOleClientSite.ShowObject() {}
 void IOleClientSite.OnShowWindow(bool fShow) {}
 void IOleClientSite.RequestNewObjectLayout() {}

 #endregion

 #region IDocHostUIHandler methods

 public uint ShowContextMenu(uint dwID, ref tagPOINT ppt, object pcmdtReserved, object pdispReserved)
 {
  //const int MenuHandled = 0; // uncomment this line, and comment the next one, to show the context menu
  const int MenuNotHandled = 1;
  return MenuNotHandled;
 }

 public uint TranslateAccelerator(ref tagMSG lpMsg, ref Guid pguidCmdGroup, uint nCmdID)
 {
  //const int KillAccelerator = 0;
  const int AllowAccelerator = 1;
  return AllowAccelerator;
 }

 public object GetDropTarget(ref object pDropTarget) 
 {
  return pDropTarget;
 }

 public object GetExternal()
 {
  return null;
 }

 public uint TranslateUrl(uint dwTranslate, string URLIn, ref string URLOut)
 {
  //const int Translated = 0;
  const int NotTranslated = 1;
  return NotTranslated;
 }

 public IDataObject FilterDataObject(IDataObject pDO) 
 {
  return null;
 }

 public void GetHostInfo(ref DOCHOSTUIINFO theHostUIInfo) {}
 public void ShowUI(uint dwID, ref object pActiveObject, ref object pCommandTarget, ref object pFrame, ref object pDoc) {}
 public void HideUI() {}
 public void UpdateUI() {}
 public void EnableModeless(int fEnable) {}
 public void OnDocWindowActivate(int fActivate) {}
 public void OnFrameWindowActivate(int fActivate) {}
 public void ResizeBorder(ref tagRECT prcBorder, int pUIWindow, int fRameWindow) {}
 public void GetOptionKeyPath(ref string pchKey, uint dw) {}

 #endregion

 #region IOleDocumentSite methods

 public void ActivateMe(ref object pViewToActivate) {}

 #endregion

 #region IDocHostShowUI methods

 public uint ShowMessage(IntPtr hwnd, string msgText, string msgCaption,
  uint dwType, string lpstrHelpFile, uint dwHelpContext,
  out int lpResult)
 {
  // msgText is the contents of the MessageBox
  // msgCaption is the text on the MessageBox caption bar

  // this is where you can deal with javascript popup boxes and simulate clicking different buttons
  lpResult = (int) DialogBoxCommandId.Ok;

  // return one of these values
  const int MessageHandled = 0;    // MsHtml won't display its MessageBox
  //const int MessageNotHandled = 1; // MsHtml will display its MessageBox
  return MessageHandled;
 }
                                  
 public uint ShowHelp(IntPtr hwnd, string pszHelpFile, uint uCommand, uint dwData, 
  tagPOINT ptMouse, object pDispatchObjectHit)
 {
  // pDispatchObject will reference an object of
  // type mshtml.HTMLDocumentClass, or other class
  // representing something on the HTML page.
      
  // return one of these values
  //const int HelpHandled = 0;    // MsHtml won't display its Help window
  const int HelpNotHandled = 1; // MsHtml will display its Help window

  return HelpNotHandled;
 }
                      
 #endregion

 #region IDispatch methods

 [DispId(-5512)] // DISPID_AMBIENT_DLCONTROL = -5512
 public int IDispatch_Invoke_Handler()
 {
  // I have found that DLCTL_NO_CLIENTPULL means that things like meta refreshes are not executed
  int lReturn = (int) (ComConstants.DLCTL_PRAGMA_NO_CACHE | ComConstants.DLCTL_SILENT | ComConstants.DLCTL_NO_BEHAVIORS | ComConstants.DLCTL_NO_JAVA | ComConstants.DLCTL_NO_FRAMEDOWNLOAD);
  if (!Configuration.LoadActiveXControls)
  {
   lReturn |= (int) (ComConstants.DLCTL_NO_DLACTIVEXCTLS | ComConstants.DLCTL_NO_RUNACTIVEXCTLS);
  }
  if (Configuration.LoadImages)
  {
   lReturn |= (int) ComConstants.DLCTL_DLIMAGES;
  }
  return lReturn;
 }

 #endregion
}
```

In my code, I have a member variable mWebBrowser, of type AxWebBrowser, inside my WebBrowser class. After I've created it, the following lines are necessary to "register" the WebBrowser class as the host:

``` csharp
IOleObject lOleObject = (IOleObject) mWebBrowser.GetOcx();
lOleObject.SetClientSite(this);
```