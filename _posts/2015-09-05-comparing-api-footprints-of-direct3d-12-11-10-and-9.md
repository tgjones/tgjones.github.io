---
title:   "Comparing API footprints of Direct3D 12, 11, 10, and 9"
date:    2015-09-05 16:19:00 UTC
---

I've spent some time during the last few days reading through the [Direct3D 12 documentation](https://msdn.microsoft.com/en-us/library/windows/desktop/dn903821(v=vs.85).aspx). I consider myself to be quite familiar with Direct3D 11, but Direct3D 12 is going to take some getting used to. As Microsoft itself [says](https://msdn.microsoft.com/en-us/library/windows/desktop/dn899228(v=vs.85).aspx):

> Direct3D 12 is certainly for advanced graphics programmers, it requires a fine level of tuning and significant graphics expertise.

I feel confident in predicting that Direct3D 12 is mostly going to be used by engines. If you're writing a game, and not using an existing engine, the recommended approach seems to be to continue using Direct3D 11 - the new rendering features in Direct3D 12 (conservative rasterization, etc.) have also been added to Direct3D 11.3, so you won't be missing out.

Anyway, what prompted this blog post was [this line](https://msdn.microsoft.com/en-us/library/windows/desktop/dn899228(v=vs.85).aspx):

> Another advantage that Direct3D 12 has is its small API footprint. There are around 200 methods, and about one third of these do all the heavy lifting.

It made me wonder - just how big are the API footprints of previous versions of Direct3D? So I made a list of all the functions / methods in Direct3D 9, Direct3D 10, Direct3D 11, and Direct3D 12, and then graphed it:

![](/assets/posts/d3d-versions-function-count.png)

So Direct3D 9 is the "winner" with 201 functions / methods. Then Direct3D reset the clock, so-to-speak, with 142. Direct3D 11 increased that to 175. And Direct3D 12, as promised, reduces the API surface area to 126 functions / methods.

Unfortunately, of course, there is no correlation between ease of use and API size!

---

A number of caveats to the data I used to generate this graph:

* I'm just comparing functions and interface methods, not structures, enumerations, etc.
* Direct3D 9 included a swap chain API. In Direct3D 10, this was moved to the separate DXGI API, which I haven't counted.
* I only looked at the first release of each version - so not Direct3D 9Ex, not Direct3D 10.1, etc.
* Only interfaces that have unique methods are included.
* I have only included Core and Resource APIs, where applicable. I have not included debug, shader, or effect APIs, or other helper functions.

Here is the raw data I used:

* [Direct3D 9 functions / methods](/assets/55eb14b5f51f2775e8000003/D3D9.txt)
* [Direct3D 10 functions / methods](/assets/55eb14b5f51f27c642000005/D3D10.txt)
* [Direct3D 11 functions / methods](/assets/55eb14b5f51f2775e8000004/D3D11.txt)
* [Direct3D 12 functions / methods](/assets/55eb14b5f51f27c642000006/D3D12.txt)