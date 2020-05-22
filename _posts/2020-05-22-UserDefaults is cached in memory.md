---
layout: post
title:  "UserDefaults is cached in memory"
date:   2020-05-22 22:57:00 +0200
---

`UserDefaults` caches its contents in memory, and loads it on application startup even if it's not used. This is vaguely documented.
From the [docs](https://developer.apple.com/documentation/foundation/userdefaultss):

>UserDefaults caches the information to avoid having to open the user’s defaults database each time you need a default value.

Let's do an experiment to confirm that.

Let's make a very simple app based on the single view app template. 
Before doing anything, let's record how much memory does our app consumes before any `UserDefaults`-related work.

On the iPhone SE 2 simulator: 8.7 MB.

Good. Let's save 100 MB of data to `UserDefaults` and see what we get.

```swift
func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
    UserDefaults.standard.set(Data(repeating: 1, count: 100_000_000), forKey: "data")    
    return true
}
```

After running that I got: 104 MB.

OK, let's run again removing that code; essentially like our very first run, and see if anything changes.

Now we got: 104 MB.

So, even if we no longer save or read from `UserDefaults`, just having that data there makes our app consumes what that data worth in memory.

[Code for the experiment](https://github.com/ahmedk92/UserDefaultsCaching).