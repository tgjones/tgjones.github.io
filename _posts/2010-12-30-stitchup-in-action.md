---
title:   "StitchUp in action"
date:    2010-12-30 11:40:00 UTC
---

After launching an [open source project](/blog/archive/2010/11/13/introducing-stitchup-generating-shaders-from-hlsl-shader-fragments), it's always gratifying to see it being used by other people. So far, I know about two instances where StitchUp has been integrated into a graphics / game engine.

## MadoxLabs Framework

* [MadoxLabs Xbox Framework - StitchUp](http://madox.ca/mediawiki/index.php/XBox:Framework:StitchUp)

That link is a great write-up of how StitchUp was integrated into their engine. The write-up concludes with this note on performance:

> All of the extra function calls, structure manipulating and variable copying that StitchUp does seems like a lot of work on the surface. To look into this, I compared the generated ASM file for a StitchUp shader and my original hand written shader. The result is that the StitchUp shader is smaller or equal to the original shader. In two places, the StitchUp shader avoided an if statement. In one place, it avoided an rsq call. There is also less preshader blocks in the ASM files. In all, the StitchUp shader is about 17 instructions less.

This version includes conditionals as part of the fragment syntax; this is useful, and I hope to incorporate it into the official version of StitchUp.

## Engine Nine

* [Engine Nine](http://nine.codeplex.com/)

I hadn't heard of this engine before, but it looks promising [especially now that it includes StitchUp ;-)]. I've had a browse through the source code, and the StitchUp integration looks quite slick.