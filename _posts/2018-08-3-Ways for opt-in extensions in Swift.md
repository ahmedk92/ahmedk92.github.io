---
layout: post
title:  "Ways for opt-in extensions in Swift"
date:   2018-08-03 09:59:00 +0200
---

Suppose you want to add a quick facility for `UIViewController` subclasses to show a confirmation `UIAlertController`. First thought would probably be like this:

```swift
extension UIViewController {
    func showConfirmationAlert(okAction: @escaping (() -> ()), cancelAction: @escaping (() -> ())) {
        let alert = UIAlertController(title: NSLocalizedString("Are you sure?", comment: ""), message: nil, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: NSLocalizedString("Yes", comment: ""), style: .default, handler: { (action) in
            okAction()
        }))
        alert.addAction(UIAlertAction(title: NSLocalizedString("No", comment: ""), style: .cancel, handler: { (action) in
            cancelAction()
        }))
        self.present(alert, animated: true, completion: nil)
    }
}
```

Good. But the problem here is that our extension is available (and visible) to every `UIViewController` subclass in our application. Some would see nothing wrong in this, but others would find this being an "API pollution". They would prefer an opt-in alternative. Funnily, in Objective-C you had to import the `.h` file where the category is defined; making it a perfect way for opt-in purposes.

Now, back to Swift. One solution would be the following:

```swift
protocol AlertConfirming {
    func showConfirmationAlert(okAction: @escaping (() -> ()), cancelAction: @escaping (() -> ()))
}

extension AlertConfirming where Self: UIViewController {
    func showConfirmationAlert(okAction: @escaping (() -> ()), cancelAction: @escaping (() -> ())) {
        let alert = UIAlertController(title: NSLocalizedString("Are you sure?", comment: ""), message: nil, preferredStyle: .alert)
        alert.addAction(UIAlertAction(title: NSLocalizedString("Yes", comment: ""), style: .default, handler: { (action) in
            okAction()
        }))
        alert.addAction(UIAlertAction(title: NSLocalizedString("No", comment: ""), style: .cancel, handler: { (action) in
            cancelAction()
        }))
        self.present(alert, animated: true, completion: nil)
    }

}
```