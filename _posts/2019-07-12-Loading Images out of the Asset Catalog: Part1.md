---
layout: post
title:  "Loading Images out of the Asset Catalog: Part 1"
date:   2019-07-12 14:16:00 +0200
---

## Asset Catalog
We use [Asset Catalog](https://developer.apple.com/library/archive/documentation/Xcode/Reference/xcode_ref-Asset_Catalog_Format/index.html) to store images we use inside our apps. If you're not using it, you should use it. It's not just a fancy way to organize or resources. It also does some optimizations we happen to overlook them.

However, if you have to not use Asset Catalogs, there are some things you should be aware of. In this part, we are going to investigate the **scale** property of an image.

## @1x, @2x, @3x

The most noticeable feature of asset catalogs is the facility to provide different versions of an image for different screen scales. If we don't use asset catalogs, we are required to put such information in the bundled file's name (e.g. pic@2x.png, pic@3x.png). If we don't provide such info, an `UIImage` instance created from such file will have a default scale factor of 1.0.

Good, but what if we have an app that loads images from a folder? We don't want to hard-code image names in our project. We had this use case in one of our apps, and the code to read such images involved enumerating file paths under the given folder URL and creating `UIImage` instances like this:

```swift
for path in paths {
    let image = UIImage(contentsOfFile: path)
}
```

We also didn't have scale modifiers in the file names. So, every `UIImage` loaded with a scale factor of 1.0.

## What is the implication?

From Apple's docs on the [`scale`](https://developer.apple.com/documentation/uikit/uiimage/1624110-scale) property of the `UIImage` class: 

> If you multiply the logical size of the image (stored in the size property) by the value in this property, you get the dimensions of the image in pixels.

I'm not so good at English, but I don't think that phrasing is the best. The way it's put makes you think that the pixel size is the variable here. It isn't. The pixel size is fixed. The logical size of the image (what you get via the `size` property) is the pixel size divided by the scale factor (of the image; not the screen).

## Again, what is the implication?

If you have a `UIImageView` that derives its size from its content, and we want it to always show the full pixel size of its content, ignoring choosing the appropriate scale factor leads to incorrect frame size. For example, if you have an image that is 640x640 pixels. What logical size should a self-sizing `UIImageView` have in a @2x screen? The answer is 320x320 points. But loading such image without providing a scale factor leads to having an image view with a logical size of 640x640 points. That is, 1280x1280 pixels. That's double of the actual size. Which means that the image is upscaled. Which affects perceived quality, or generally, not the wanted actual pixel size.

So, instead, we should create `UIImage`s using [init(data:scale:)](https://developer.apple.com/documentation/uikit/uiimage/1624109-init) like this:

```swift
let image = UIImage(data: data, scale: UIScreen.main.scale)
```

Where `data` is a `Data` instance created from a given file path or a URL.

This way we ensure the image appears at its actual pixel size in any device.
