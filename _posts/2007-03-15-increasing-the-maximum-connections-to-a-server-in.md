---
title:   "Increasing the maximum connections to a server in IE and Firefox"
date:    2007-03-15 16:12:00 UTC
---

Although the HTTP 1.1 specification says that only two connections to a single server should be allowed through a browser, it's often necessary (well, preferable) to open more connections. For example, when downloading multiple files, or simply to increase rendering speed.

In Internet Explorer, you just need to add the following value to your registry:

```
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings]
"MaxConnectionsPerServer"=dword:00000010
```

In Firefox, type about:config in the address bar, and then change this value, for example to 16:

```
network.http.max-persistent-connections-per-server
```

Hope this helps!