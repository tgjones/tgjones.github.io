---
title:   "Vista - Failed to install ISKernel files"
date:    2006-12-29 01:29:00 UTC
---

I've hit my first real snag with Vista - while trying to install Macromedia Contribute 3, I got this error: "1: Failed to install ISkernel files. Make sure you have appropriate privileges on this machine."

<a onblur="try {parent.deselectBloggerImageGracefully();} catch(e) {}" href="http://bp1.blogger.com/_Wywrr60_Hxc/RZRxExDEjAI/AAAAAAAAAAM/uiJx_JNnDEg/s1600-h/Screenshot.jpg"><img style="cursor: pointer;" src="http://bp1.blogger.com/_Wywrr60_Hxc/RZRxExDEjAI/AAAAAAAAAAM/uiJx_JNnDEg/s400/Screenshot.jpg" alt="" id="BLOGGER_PHOTO_ID_5013756611869838338" border="0" /></a>

Apparently you can get that if you try to run a .msi directly, which I was. If you use a setup.exe bootstrapper, there's no problems, but I didn't have one.

The simple answer, I discovered, is to build yourself one. All you need to do is create an app, I did mine in VB (because I accidentally clicked that project type instead of C#, I hasten to point out) but you could use anything, that simply starts the .msi process. Then build your app, and right-click on your exe in Explorer and choose "Run this program as an administrator" on the Compatibility tab. Hey presto!<div style="clear:both; padding-bottom:0.25em"></div>