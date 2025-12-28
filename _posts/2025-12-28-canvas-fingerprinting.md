---
layout: post
title: "Canvas fingerprinting"
---
The HTML \<canvas\> element can be used to draw graphics. To use it, JavaScript scripts select the canvas and use canvas methods on it. Here is example usage which draws a circle:

```html
<script>
var c = document.getElementById("myCanvas");
var ctx = c.getContext("2d");
ctx.beginPath();
ctx.arc(95, 50, 40, 0, 2 * Math.PI);
ctx.stroke();
</script>
```


This is usually used by websites for image manipulation, games, or charts. So, it may be surprising that this seemingly unrelated functionality can be exploited to implement storageless online tracking, called *canvas fingerprinting*. 

There are minute differences in how images may be rendered on a screen, tied to a specific user. Several factors control image rendering:
- Browser
- Font engine
- Operating system
- GPU/CPU architecture
	- Floating-point precision differences
	- Manufacturing variation even in identical hardware units, that cause timing or clock differences
- Graphics drivers
	- Vendor optimizations
	- Anti-aliasing algorithm default differences

Using JavaScript, websites can draw high-entropy, invisible images that amplify these differences, and then create an identifier from the sequence of raw pixels generated. These fingerprinting scripts may draw multiple lines of text in common fonts, emojis, overlapping shapes/gradients, and/or shadows. The scripts may also check for browser support for certain features, like unicode or specific fonts, to discover more differences.

In the seminal paper *The Web Never Forgets: Persistent Tracking Mechanisms in the Wild*, Acar et al. (2014), the most popular canvas fingerprinting script found amongst the top 100000 most crawled websites did exactly this:
- Draw the text `Cwm fjordbank glyphs vext quiz` (a perfect pangram) twice with diﬀerent colors 
- Check if Unicode is supported by the browser by printing the character U+1F603.
- Check if the canvas method `globalCompositeOperation` is supported by the browser
- Draw two rectangles and check if a speciﬁc point is in the current canvas path by calling the canvas method `isPointInPath`

After drawing fingerprinting images, the JavaScript methods `toDataURL()` and/or `getImageData()` can be used to get the raw pixel data, and a hash of this string can be created as the final identifier. Even single-pixel differences will generate different hashes. Combined with other fingerprinting techniques, it is likely it can lead to truly unique identifiers.
