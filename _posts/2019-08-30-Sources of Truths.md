---
layout: post
title:  "Sources of Truths"
date:   2019-08-30 16:06:00 +0200
---

It's not uncommon to see a variable in some codebase that looks like this:

```swift
var isAlertShown: Bool
```

Or:

```swift
var pageIndex: Int
```

Such variables are used in order to track some **state**. 
Let's see an example, a contrived one: 

We are going to show an alert whenever the app receives a remote notification. But we want to avoid attempting to present a new one if an alert is already shown. 
So we are going to use a `UIAlertController` for this task. 
One may write code like this:

```swift
class ViewController: UIViewController {
    private var isAlertShown = false

    private func showAlert() {
        guard !isAlertShown else { return }
        let alert = UIAlertController(title: "New Message!", message: nil, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .cancel, handler: { (_) in
            self.isAlertShown = false
        }))
    
        isAlertShown = true
    
        present(alert, animated: true, completion: nil)
    }
}
```

Here we're using `isAlertShown` to track the state telling if our alert is shown or not. 
We have to **mutate this variable correctly to ensure it correctly represents the actual state of whether the alert is shown or not**.
However, if the alert got dismissed by any other mean than our cancel action, the variable `isAlertShown` will have a wrong value.

The reason is `isAlertShown` is not the *source of truth*. We can avoid such bug if we *queried* a fresh value from the ever-changing environment itself. 
For this particular example, we can check if the `presentedViewController` property is of type `UIAlertController`. Like this:

```swift
class ViewController: UIViewController {
    private var isAlertShown: Bool {
        return presentedViewController is UIAlertController
    }

    private func showAlert() {
        guard !isAlertShown else { return }
        
        let alert = UIAlertController(title: "New Message!", message: nil, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .cancel, handler: nil))

        present(alert, animated: true, completion: nil)
    }
}
```

Alternatively, we can use a weak reference if we want to be sure that the presented alert is a particular one.

```swift
class ViewController: UIViewController {
    private weak var shownAlert: UIAlertController?

    private func showAlert() {
        guard shownAlert == nil else { return }
        
        let alert = UIAlertController(title: "New Message!", message: nil, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: "OK", style: .cancel, handler: nil))

        shownAlert = alert

        present(alert, animated: true, completion: nil)
    }
}
```

Once it's dismissed, by any means, the weak reference will be nil. No need for mutations.

## Conclusion
So, I think the idea is clear now.
Want to track the download status of some book? **Infer** it from the existence of the relevant files.
Want to track the current index of a paged `UICollectionView`? **Infer** it from the relation of the `contentOffset` to the `contentSize`.

This principle is discussed on the [web](http://google.com/search?q=single+source+of+truth). However, it's more data (or database) oriented.
So, I wanted to discuss it in a close-to-UI context, where we, iOS developers, spend a significant time.

Thanks for reading!
