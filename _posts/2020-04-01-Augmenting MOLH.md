---
layout: post
title:  "Augmenting MOLH"
date:   2020-04-01 10:34:00 +0200
---

[MOLH](https://github.com/MoathOthman/MOLH/) is a popular library among developers whose audience are of Arabic background. It's common for Arabic apps to support multiple languages, and also **support changing between them without restarting**. I wrote [here]({{ site.baseurl }}{% post_url 2017-12-15-Acheiving Dynamic Localisation in iOSPagination %}) how iOS doesn't support that out-of-the-box and what were the possible workarounds back-then (this was a write-up for a solution I reached before MOLH was made public).
In this article I'll try to briefly reiterate how MOLH works to reach to its limitation, and then try to find some solutions/workarounds.

Before we start, these limitations are not specific to MOLH; it's rather general to any approach that tries to work around iOS's limitation to change the app's language without restarting.

## How MOLH works

(Feel free to skip this section if you already know how MOLH works)

Two aspects of an app reflects to the user that it respects the desired localization: the language of the texts, and the direction of how the flow of the views (left-to-right or right-to-left).
As you may already know, the default behavior of iOS is to pick and suitable language at startup (of the app) and upon that it picks the relevant strings from string files in that language's `lproj` folder, as well as applying the corresponding language direction, **and sticks to that**.

Two goals here:
1. Guiding `NSLocalizedString` to search the new language folder (i..e `ar.lproj`, `en.lproj`, ...etc)
2. Forcing new views to flip (or not) with respect to the new language.


### (1) NSLocalizedString
`NSLocalizedString` delegates to `NSBundle.mainBundle` for getting the suitable localized string value.
Upon app startup, `NSBundle.mainBundle` makes up its mind **once and for all** on what language folder it's going to search till the next run. To overcome this limitation, MOLH [swizzles the main bundle](https://github.com/MoathOthman/MOLH/blob/313691443043f0da83502040f39b852cd9e3e0e8/Sources/MOLH/MOLH.swift#L184) and dynamically loads the relevant bundle to search for the required string.

### (2) Flipping views
This is somewhat easier. `UIView` formally supports forcing the language direction independently of the actual app language. This is done via setting the [`semanticContentAttribute`](https://developer.apple.com/documentation/uikit/uiview/1622461-semanticcontentattribute) to the desired value.
MOLH does this for all newly created views by [using the appearance proxy to set the suitable `semanticContentAttribute`](https://github.com/MoathOthman/MOLH/blob/313691443043f0da83502040f39b852cd9e3e0e8/Sources/MOLH/MOLH.swift#L133). This is why it needs the app to virtually "restart" (i.e. start over from the root view controller, re-creating all views).

## What MOLH doesn't solve (and it doesn't have to)

### Formatters
There are some classes that must pick some locale information to correctly present its data. Such classes include formatters (e.g. `NumberFormmatter` and `DateFormatter`). If we don't explicitly set the `locale` property of such formatters to the desired language, it will pick the actual app language, causing a date to be displayed in the wrong language for example. 

### NSTextAlignmentJustified
Formatters are easy to handle as we saw. However, justified `UILabel` and `UITextView` yield unwanted results. This is because justifying works by distributing space in every line of text so that each line starts and ends at the same start and end points respectively, **and aligning the remaining last line either left or right if it's not wide enough**. As you you may already guessed, left or right alignment is picked according to the actual app language.

![Incorrect Justify Last Line]({{site.url}}/assets/incorrectjustify.png)

Unlike formatters, it seems to be no explicit way to guide `UILabel` nor `UITextView` on how to force that alignment. However, there's a way to achieve the desired justifying with `NSAttributedString`.
`NSAttributedString` (a world of its own) accepts and attribute called `paragraphStyle`. 
What matters to us from it is the [`baseWritingDirection`](https://developer.apple.com/documentation/uikit/nsmutableparagraphstyle/1534601-basewritingdirection?language=objc) property. We can set it to either to `NSWritingDirectionLeftToRight` or `NSWritingDirectionRightToLeft`. This affects decisions that rely on such information, namely justifying and natural alignment. Sample code:

```swift
let attributedString = NSAttributedString(string: string, attributes: [
    .paragraphStyle: {
        let style = NSMutableParagraphStyle()
        style.alignment = .justified
        style.baseWritingDirection = .rightToLeft
        return style
    }()
])
```

## Demo
I made a demo of these problem (without the solutions). You can check it [here](https://github.com/ahmedk92/FormatterDefaults). Make sure to read the usage notes before inspecting.

Thanks for reading! Corrections and suggestions are welcome.