---
layout: post
title:  "Acheiving Dynamic Localisation in iOS"
date:   2017-12-15 09:00:00 +0200
---

While you can make your app support multiple languages, still there's no easy (and documented) way to dynamically change the app's localisation. You either let the user chose the iOS device's language; and your app just follows along. Or you can use that [AppleLanguages solution](https://stackoverflow.com/q/1669645/715593). But if you want the change somewhere after the app has started (i.e. not in the `main` function), you have to close the app, and wait for the user to open it again. Which is not the best we can expect.

## Alternatives
There are many libraries (on cocoapods for example) that enables you to acheive that without closing the app. But they just work on strings; either an `NSLocalizedString` replacement, or overriding its behaviour. Which is good, but unfortunately ignore flipping views on RTL languages like Arabic.

### What worked for me
To organize, we have two problems: (1) flipping views based on the chosen language direction preference. (2) Guiding `NSLocalizedString` to use the correct strings.

### (1) Flipping views
Since iOS 9, there's this property that enables you to decide, for a view, whether it should be always displayed left-to-right, or right-to-left (despite whatever is the language), or it should follow the language. The property is [semanticContentAttribute](https://developer.apple.com/documentation/uikit/uiview/1622461-semanticcontentattribute).  

Now, the question is when to set that property to take its effect? and how? Usually, UI is diversely created; maybe from storyboards, XIBs, or code. So, an early point of initialization is a good time; e.g. `initWithCoder:` (for views designed in Interface Builder) or `iniWithFrame` (for views created by code). `awakeFromNib` is a good time too for views designed in Interface Builder. OK, but how would we override those methods? [Swizzling](http://nshipster.com/method-swizzling/) to the rescue.

Through swizzling, we can check the language in `awakeFromNib` for example, and set the relevant `semanticContentAttribute`. Good!, but there may be a little caveat. There may be some views that are required to be always left-to-right or righ-to-left all the time; we don't want to mess them up. We can avoid that by a simple check; we only apply the `semanticContentAttribute` to views with an `.unspecified` value.

So far so good. But what about currently visible views? what can we do about them? One can think of making every visible view observe the language change (via `NSNotificationCenter` for example), then try to force a redraw. But I don't like this approach, and I prefer to simply restart the flow of my app. That is re-assigning the window's root view controller with a new one, and start over the app's flow. Something like this:

```swift
for view in UIApplication.shared.window.rootViewController.view.subviews {
    view.removeFromSuperView()
}
UIApplication.shared.window.rootViewController = UIApplication.shared.window.rootViewController.storyBoard.instantiateInitialViewController 
```
That's it. And it works nicely.

### (2) NSLocalizedString
As mentioned above, there are many libraries that do this. Most of them offer a replacement for `NSLocalizedString`. But sometimes, the dynamic localiztation feature is required so late in the project life-time that `NSLocalizedString` is scattered all over the project. So, overriding its behavior would be better. I heard about a library that does this, but couldn't find it 😑. However, that overriding is not hard.

I built my solution on [this StackOverflow answer](https://stackoverflow.com/a/20257557/715593). The whole discussion is useful. The idea is that `NSLocalizedString` is just a macro that uses `localizedStringForKey:value:table:` method of the `NSBundle` class; specifically on `[NSBundle mainBundle]` instance. So, swizzling is again used here. In short, the `[NSBundle mainBundle]` instance's implementation is replaced by an `NSBundle` subclass that, on language change, makes the `[NSBundle mainBundle]` return a new bundle that is relevant to the selected language. Easy, but tricky.

That's it.

### Notes
- Maybe it won't be enough to just have the views flipped; you may want text views (e.g. `UILabel`, `UITextField`) to flip the text alignment too. That's acheivable in our same `awakeFromNib` implementation explained above. All you need is checking wether the current view is a `UILabel`, then repeat the same logic; check wether the text alignment (`NSTextAlignment`) is set to `.natural`, then set to the correct alignment (i.e. `.left`, or `.right`).


