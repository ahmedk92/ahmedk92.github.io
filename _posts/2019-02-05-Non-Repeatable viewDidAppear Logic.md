---
layout: post
title:  "Non-Repeatable viewDidAppear Logic"
date:   2019-02-05 03:40:00 +0200
---

Sometimes we need to show an alert, apply a gradient, or conditionally show another view controller on the startup of a view controller. We wish to do such thing in `viewDidLoad`, however we end up doing it in `viewDidAppear(_:)`. This is because such purposes have requirements that are not fulfilled when `viewDidLoad` executes; e.g. frames are not yet correct, view hierarchy is not ready, ...etc. 

One annoyance with `viewDidAppear(_:)` is that it can get called multiple times. For example, if you present a new view controller, then dismiss it sometime after, it gets called again. You have to handle such logic that shouldn't be repeated.

## Solutions that work, but I don't quite like

### 1. Booleans

One can introduce a `Bool` like `viewAppeared`; checking and setting it for once:

```swift

var viewAppeared = false

override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)

    if !viewAppeared {
        // Do non-repeatable logic...
    }

    viewAppeared = true
}
```

It works. However, I tend to try to avoid "state variables" as much as I can (doesn't mean I succeed much 😅). Although they look simple, bugs find their way around them. And in this particular situation, you may need to repeat some check you did earlier in `viewDidLoad`, and it gets less nice:

```swift
var name: String?

override func viewDidLoad() {
    super.viewDidLoad()

    if let name = name {
        // conditional initial setup 
    }
}

override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)

    if !viewAppeared {

        // this check again
        if let name = name {

        }
    }

    viewAppeared = true
}

```

### 2. Deferring with GCD

Magic. GCD helps in executing code *later*. There are two ways to do it for our need: `async` and `asyncAfter(deadline:execute:)`. We can use either in `viewDidLoad` to execute non-repeatable code "later enough".

`asyncAfter(deadline:execute:)` is straight forward:

```swift

override func viewDidLoad() {
    super.viewDidLoad()

    DispatchQueue.main.asyncAfter(deadline: .now() + 0.1) {
    // Non-repeated logic
    }
}
```

Why 0.1 seconds? No idea. It's just late enough. It works, but it depends on a magic number, we don't know exactly when this code runs.

What about just `async`? It works too, no explicit delay needed. That's because code in a `DispatchQueue.main.async` block **is executed in the next [run loop](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)**.

Believe me, I've read a lot about run loops, I still don't understand them much. However, for the main thread, you can interpret them as time slices in which the main thread accepts input, updates UI, calculate layouts, and more importantly ***"polls"* code enqueued via `DispatchQueue.main.async`**.

So, When we dispatch code async on the main queue from the main thread, it doesn't run immediately; it waits to be polled in the next run loop, leaving enough time for the requirements mentioned above to be fulfilled. Remember our blog post? 😂 Now let's continue it. This GCD thing worth it's own blog post.

>If you look at code I wrote (I hope you don't), you'll find me guilty of using this to hack my way through. It's not good, I don't recommend it. We should invest more time to really solve latency problems rather than working around it.

## A Better Solution?

Here I suggest a solution that I think is better. GCD gave us a hint, we need to queue tasks on `viewDidLoad`; but execute it later. So, what about queuing it on `viewDidLoad`, then execute it on `viewDidAppear(_:)`?

```swift
private var viewDidAppearQueue: [() -> ()] = []

override func viewDidLoad() {
    super.viewDidLoad()
        
    viewDidAppearQueue.append {
        // Our non-repeatable logic
    }
}
    
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)
        
    // Dequeue tasks and execute them in FIFO order
    while !viewDidAppearQueue.isEmpty {
        viewDidAppearQueue.removeFirst()()
    }
}
```

We introduce a simple array of closures. We add our tasks as closures in `viewDidLoad`. Then in `viewDidAppear(_:)` we execute each closure and remove it from the array. If there are no tasks; nothing happens, just what we need.

We also don't need to weakly capture `self` (i.e. `[weak self] in`) when appending closures. This is because strong references to the closures are lost when we remove them from the array. So, we should be safe. 

Thanks for reading. Feedback is welcome.

