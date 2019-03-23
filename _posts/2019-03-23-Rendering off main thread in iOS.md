---
layout: post
title:  "Rendering off Main Thread in iOS"
date:   2019-03-23 11:26:00 +0200
---

One of the first lessons we learn in iOS development is that UIKit classes (`UILabel`, `UIImageView`, ...etc) shouldn't be touched outside the main thread. Sometimes we learn it [the hard way](https://stackoverflow.com/a/12927485/715593). However, **this doesn't mean we cannot do any form of rendering off the main thread**.

Classes like `NSAttributedString` and `UIImage` come with [methods](https://developer.apple.com/documentation/foundation/nsattributedstring/1531631-draw) for drawing to a given graphics context; an [image context](https://developer.apple.com/documentation/uikit/1623912-uigraphicsbeginimagecontextwitho?language=objc) for our use case. This doesn't mandate being done in a particular thread. Not only this, but UIKit enables us to export what's drawn to the current context to a bitmap image using [`UIGraphicsGetImageFromCurrentImageContext`](https://developer.apple.com/documentation/uikit/1623924-uigraphicsgetimagefromcurrentima). This means we can do any complex drawing like we do with CoreGraphics in `drawRect:`, and then export this to a bitmap.

All what we have to do next is to display the resulting image in our view. This is easily achieved by setting the views [`layer.contents`](https://developer.apple.com/documentation/quartzcore/calayer/1410773-contents) property to a `CGImage` representation of our image. And that's it.

## Cool, but why?

UIKit performance is great 99% of the time. However, it's not the best we can acheive. UIKit performance degrades noticebly when rendering large scrolling amounts of text and images with varying sizes, in addition to relying on AutoLayout for sizing. [AutoLayout came along way in iOS 12](https://developer.apple.com/videos/play/wwdc2018/220), but earlier iOS versions are still in support, and no matter how fast AutoLayout becomes, it still works on the main thread.

AutoLayout is not only the slowing factor. Actual rendering and intrinsic content size calculation also happens on the main thread. I've profiled stuttering scrolling performances and the culprit was none other than regular text drawing invoked from `UILabel`'s drawing.

## Rendering and Sizing

If you notice, we're dealing with two types of problems: (1) Rendering, i.e. the graphical content we see, and (2) Sizing, i.e. what space our rendered content will consume.

We talked about rendering methods above. If you notice, those rendering methods rely on a `CGRect` input, that is the bounding box of the graphical content. So, this implies a prior sizing step. Sizing images is usually easy; as we know beforehand where it would appear, and at what size. Text may be a bit trickier; as we usually fix a dimension (width or height) then let the text flow with respect to the desired alignment, consuming space depending on the font and other text attributes. Fortunately, there are more than one way to calculate bounding rectangles for attributed strings. The simplest method is `NSAttributedString`'s [boundingRect](https://developer.apple.com/documentation/foundation/nsstring/1524729-boundingrect). Other ways involve utilizing `NSLayoutManager`, `NSTextContainer`, and `NSTextStorage` trio for advanced text layout.

## A Simple Demo

I made a very [simplistic demo](https://github.com/ahmedk92/PrerenderingDemo) that showcases the gains in a scrolling use case. Notice the regular implementation (left) stutters on fast scrolling, while the prerendered implementation (right) scrolls like the wind. One cost is to engineer when to pre-load the pre-rendered conent. I went with an inefficient way for the sake of simplicity (i.e. loading all beforehand). This is a real cost for such approach.

## Where to go from here?

I'm just exploring this technique myself. It's not something new. There are already amazing libraries which adopt this approach; namely Ryan Nystrom's [StyledTextKit](https://github.com/GitHawkApp/StyledTextKit), and [Texture](https://texturegroup.org)(AsyncDisplayKit).


## Notes

- When using `UIGraphicsBeginImageContextWithOptions`, take some notes:
    - The third parameter is the scale at which the bitmap is generated. A zero values picks the device's scale (i.e. 2x, 3x). This helpful in emulating vector drawing behavior in zoomable views by redrawing content at a higher scale (e.g. device scale * zoom scale).
    - Don't forget to call `UIGraphicsEndImageContext` after done dealing with the image context. This is to clean up memory. This approach can use memory extensively, it's a really important step.
    - The second parameter is whether to draw opaquely or transparent. If you know that your view is going to be not transparent, it's good to set this to true, as transparency is a computationally expensive task. See this [WWDC session at 29:28 mark](https://developer.apple.com/videos/play/wwdc2012/506/).

- Use a serial `DispatchQueue` instead of a gloabl queue if you're going to render multiple views in series, or group them in a single block to be executed in a global queue. Anyways don't execute each block individually in a gobal queue. This to avoid creating more threads than CPU can handle; what's called "Thread Explosion". See this [WWDC talk at 16:42 mark](https://developer.apple.com/videos/play/wwdc2018/219/?time=1002).
