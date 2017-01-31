# Wide-Gamut Client Hints explained

## What is the problem?

Wide-Gamut cameras and displays are common in consumer hardware. Delivering wide gamut content is error prone and can result in degraded/inconsistent visual experiences and degraded web performance for legacy sRGB devices and browsers. Currently CSS4 Media Queries with `<picture>` elements is the only viable approach. The preferred solution is to utilize Client Hints to negotiate the appropriate content for the user experience.

Since the 1990s sRGB has been the assumed color space for web standards. For example, CSS3 uses 8bit sRGB colors, untagged images (lacking icc, or incompatible icc) are assumed sRGB and as is video. However, sRGB represents a limited subset of the human eye's visual spectrum while wider gamut color spaces such as DCI-P3, AdobeRGB and Rec.2020 include more of the spectrum. Consumer hardware has increasingly adopted these standards. Specifically, the iPhone 7, Galaxy S7, MacBook Pro, iMac, Surface Studio all support DCI-P3 displays. LG, Samsung, Sony and others also offer UHD displays with Rec.2020.

For example, Apple is recommending now to create 2 separate assets for Normal and Wide Color images, as in Retina vs. Non-Retina images.

Creative and operations teams utilizing wide gamut content must consider four variables.

1. color space required: wide-gamut in 8bit color, or wide-gamut in high bit (eg: 10bits per channel) color
2. end user display capabilities: sRGB, P3, Rec.2020, etc...
4. browser support for image formats that support wide-gamut high colour (currently only JPEG2000, JPEGXR & PNG) 
5. browser support for colour management

This creates 4 problems for images:

1. Delivering wide gamut images (low bit) to sRGB will result banding and clamping
2. Delivering wide gamut images (low bit) to sRGB with CSS color matching will be inconsistent 
3. Delivering wide gamut images (high bit) is browser specific and limited to PNG, JPEG 2000, JPEG XR
4. ICC stripping is a common performance optimization but is problematic for FF and Chrome when attached to a non sRGB display

### Solution requirements

To balance the requirements from creative teams and operations, the following decision tree exists:

#### 1. Maximize the experience (P3 image to P3 device)
A server has 10bit/channel images and client can display wide gamut (aka serve P3 to P3)

* If client supports jp2/jpegxr then use;
* If bandwidth permits, use png.
* Otherwise reduce color space to 8bit/channel and add p3 icc and serve webp/jpg.

#### 2. Preserve the experience (P3 image to sRGB, modern browser)
A server has 10bit/channel images but client is only sRGB. Sending 8bit/channel P3 to sRGB has poor effects and will result in banding and clamping therefore we want to reduce to 8bit/channel, color correcting to sRGB.

#### 3. Legacy identification (P3 image to sRGB, legacy browser)
A server wants to reduce bytes by stripping icc metadata. However, doing this today has inconsistent experiences with legacy browsers if a non-sRGB display is attached. Therefore including the icc is required unless you can be certain the attached display is sRGB.

editorial: Notably, Chrome does not color correct to the display gamut but instead assumes the display is sRGB. This results in a saturated visual experience. 

### Wide-Gamut with CSS4 and Media-Query4

The best solution is to serve a wide-gamut image when the user is on a wide-gamut display, and an sRGB image when the user is not on a wide-gamut display. This is just another case of responsive images, and is exactly what the element and media queries were designed to handle.
This is also why Safari has added color-gamut media query to CSS Color Level 4.

```html
<picture>
  <source media="(color-gamut: p3)" srcset="photo-wide.jpg">
  <img src="photo-srgb.jpg">
</picture>
```

However, to do this properly to address all the use cases it would more accurately look like this:

```html
<picture>
  <source media="(color-gamut: p3)" type="image/vnd.ms-photo" srcset="photo-wide.jxr">
  <source media="(color-gamut: p3)" type="image/jp2" srcset="photo-wide.jp2">
  <source media="(color-gamut: p3)" srcset="photo-wide.png">
  <source media="(color-gamut: sRGB)" srcset="photo-srgb-noicc.jpg">
  <img src="photo-srgb-icc.jpg">
</picture>
```

## Proposed Solution with Client Hints
Client Hints can simplify the delivery selection for color-gamut. UA-sniffing is possible but problematic. This will allow the client to indicate the color gamut of the display, and allow the server to respond with the contextually correct image

### `Color-Gamut` Client Hint

Add `Color-Gamut` as a client hint to express the device's attached display. This would follow the proposed Media Query 4 signaling. 

Typical CH signaling

```http
GET /index.html HTTP/1.1
Host: www.example.com

...

HTTP/1.1 200 OK
Accept-CH: Color-Gamut
```

No changes are now required in the HTML:

```html
<img src="photo.jpg">
```

Followed by request with the Color-Gamut:

```http
GET /photo.jpg HTTP/1.1
Color-Gamut: p3
Host: www.example.com
Accept: image/vnd.ms-photo, */*

...

HTTP/1.1 200 OK
Vary: Color-Gamut
Content-Type: image/vnd.ms-photo

```

The core use case behind Color-Gamut hint is to enable automatic resource selection for image assets based on display type. This will enable delivery of optimal image variant without any changes in markup.

### Why not also have a `Content-Color-Gamut` CH?
The only advantage of providing a CH response header like `Content-Color-Gamut` would be to allow stripping of icc profiles. However, this will break images outside of the browser landscape. Images saved and sent via messenger will now be missing the critical icc profile.

### What about non image use cases?

Use of the `Color-Gamut` CH will also enable the optimization of CSS files. Specifically, CSS must include wide-gamut media queries. The over downloading could be avoided with the use of `Color-Gamut` CH.


## Summary
Device Characteristics and User Agent sniffing is not possible to identify users with wide gamut displays. CSS4 Media queries enable support for image selection, requiring markup changes. Introducing `Color-Gamut` client hint would simplify the html and move the resource selection to the server. 

## Acknowledgments

Many thanks to Nick Doyle