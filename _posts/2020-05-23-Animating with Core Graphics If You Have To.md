---
layout: post
title:  "Animating With Core Graphics If You Have To"
date:   2020-05-23 12:29:00 +0200
---

Skip directly to the [code](https://github.com/ahmedk92/AnimationTechniques) if you don't have the time.

___
<br/>
If you want to do custom drawing; that is, the kind of drawing that cannot be simply achieved by composing existing UIKit classes (e.g. UILabel, UIImageView, etc...), there are two main approaches to do that. Namely, there is the Core Graphics way, and the Core Animation way.

If you tried to play with these before, you may have noticed that animating with Core Animation is easier (it's in the name as you see). This is because you build your custom view using `CALayer` subclasses (`CAShapeLayer`, `CAGradientLayer`, etc..) that have animatable properties by default.
So, it's a matter of utilizing the rich animation APIs Core Animation provides like `CABasicAnimation` and `CAKeyframeAnimation`.

However, it's not the case with the Core Graphics way. 
Just as you override `draw(_:)`, you'll draw how will your view look in a single frame, depending on what data your view has. There is no high-level component of your drawing that you can communicate with to change its state. All your drawing can be thought of (and essentially is) a single bitmap (think: like a png image).

So, any change we want to do to our view, we change the necessary underlying data, then request our view to re-draw by calling [setNeedsDisplay()](https://developer.apple.com/documentation/uikit/uiview/1622437-setneedsdisplay). That's it. There's no other way.

Perfect, now, how to animate such changes?

We have to have at least a basic understanding of how a basic animation is done.

What we consider a smooth animation can be thought of a series of frames that gradually completes a story. Each frame shouldn't provide so much change; or else we would lose the smoothness (called jankiness, jittery, jumpiness, stutter, etc..). These frames also should arrive quickly one after the other for the same purpose.
For digital displays (computer monitors, mobile phones, etc..) sampling natural motion at a rate of 60 frames per second is considered ideal. We can drop to the 30s for acceptable results. We can go up to 120 too for luxury (as in the latest iPads). However, for most app uses-cases we deal with on a daily basis, 60 frames-per-second is our target.

Out of this theory, we can come up with two requirements:

1. Our single frame shouldn't take more time than 1/60 of a second to be made.
2. Even if we generate each frame under 1/60s, we still need to synchronize with the system's refresh rate. That is, when does the system request our frame to be delivered.
This because even if our frame is generated under 1/60s, beginning frame generation just at the end of the expected frame duration will probably exceed the duration requiring, causing the system to drop that frame entirely and come for the next frame instead. If this happens enough, we'll again lose smoothness even that our drawing is fast. However, this is actually improbable with UIKit, as `setNeedsDisplay()` just marks the view to be re-drawn the next drawing cycle and not immediately.

## Enter CADisplayLink

[`CADisplayLink`](https://developer.apple.com/documentation/quartzcore/cadisplaylink) does just that. From the docs:

>A timer object that allows your application to synchronize its drawing to the refresh rate of the display.

To answer our first question, how much do we have to render a frame we can compute that with the following:

```swift
let frameDuration = displayLink.targetTimestamp - displayLink.timestamp
```

To answer our second question, we only have to provide a callback function for the display link where we update our data and then call `setNeedsDisplay()`.

What's only left is how much we should change our data suitable to that time frame.
This depends on your goal, but let's have a simple example. 
Assume we want to uniformly animate the stroke of a ring-like shape over 3 seconds.
So, let's have some idealistic assumptions, and do simple maths:

1. Assume the refresh rate along those whole 3 seconds is 60 FPS.
2. Assume all frames have equal durations.

Now, since we agree that each second should have 60 frames, then our 3-second animation should have 3x60 frames = 180 frames.
Therefore, the stroke percentage should increment by 1/180 of 360 degrees.
So, the general formula can be: 

```
current_frame_share = frame_duration / whole_animation_duration
delta = current_frame_share * target_value
current_value += delta
```

Applying this to our example:

```swift

let displayLink = CADisplayLink(target: target, selector: #selector(update))

@objc func update() {
    guard endAngle < TARGET_END_ANGLE else {
        displayLink.invalidate()
        return
    }
    
    let frameDuration = displayLink.targetTimestamp - displayLink.timestamp
    let frameDurationShareOfTotalAnimationTime = frameDuration / ANIMATION_DURATION
    let amountOfRadiansToIncrement = CGFloat(frameDurationShareOfTotalAnimationTime) * TARGET_END_ANGLE
    
    endAngle += amountOfRadiansToIncrement
    setNeedsDisplay()
}
```

[Full code](https://github.com/ahmedk92/AnimationTechniques).

## Conclusion
As you saw, you're better off going the Core Animation way if you have animation in mind. Also notice that your maths can get rapidly more complex if the animation is not linear/uniform as in our example. That is, if you want ease-in or ease-out, you'll have to figure how much frames at the start and the end of the animation will have different changes than the rest of the frames, and so on for different paces, which is easier with Core Animation with [timing functions](https://developer.apple.com/documentation/quartzcore/camediatimingfunction).