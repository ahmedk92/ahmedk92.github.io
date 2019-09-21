---
layout: post
title:  "WKWebView Horizontal Paging"
date:   2017-11-03 15:23:00 +0200
---

`UIWebView` had a helpful property named `paginationMode`. It gave you horizontal pagination out of the box; specifically using the values: `leftToRight`, and `rightToLeft`. However, `UIWebView` is going out of favor, and since iOS 8, Apple officially encourages using `WKWebView`. Unfortunately, `WKWebView` doesn't have that property nor its equivalent out of the box.

Thankfully, horizontal pagination (left and right) is doable using CSS. CSS has a helpful property called `column-width`. Using that results in segmenting the html body into columns of the specified width. So, the idea is as follows:
1. Set the `column-width` property of the body to the webview's width.
2. Set the `height` property of the body to the webview's height.
3. Set `isPagingEnabled` of the webview's `scrollView` to true.

That's it (for left to right content though). It's up to you where to set the above values. For example, you can do it in `webView(_:didFinish:)` of the webview's `navigationDelegate`.

### Now, RTL support.

It's as simple as setting the `direction` style property of the html tag to `rtl`. But if that somehow badly affects your page, then you have to get more creative. One crazy solution is as follows:
1. Wrap all the contents of the `<body>` tag in one container `div`.
2. Set the `-webkit-transform` property of that div to `scale(-1, 1)`. (i.e. this results in horizontal mirroring)
3. Similarily, mirror the webview itself. (i.e. `webview.transform = CGAffeineTransform(scaledX: -1, y: 1)`)

### Credits:
Thanks to my friend and colleague [Sayed Arfa](https://www.linkedin.com/in/sayed-arfa/) for suggesting `column-width` and overall inspiration.

# Update

At the time of writing this post, I somehow missed this [thread](https://github.com/readium/architecture/issues/10). They suggested to use the undocumented(?) CSS property value `overflow:-webkit-paged-x`. This is more convenient than what's explained above as it doesn't need us to compute a width nor do additional work for RTL handling. However, it doesn't seem reliable, as the chromium project seems to [plan to remove it](https://www.chromestatus.com/feature/5731653806718976).