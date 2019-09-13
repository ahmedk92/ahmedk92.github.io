---
layout: post
title:  "translatesAutoresizingMaskIntoConstraints"
date:   2019-09-13 22:34:00 +0200
---

Did you ever forget to set `translatesAutoresizingMaskIntoConstraints` to `false` before adding constraints to a view created in code, and wasted much time in debugging? You are not alone, many of us did. 

But did you ever forget to set it to `true`, and it also did cause other troubles? This is what this article is about.


### What is `translatesAutoresizingMaskIntoConstraints`?

From the docs:

> If this property’s value is true, the system creates a set of constraints that duplicate the behavior specified by the view’s autoresizing mask. This also lets you modify the view’s size and location using the view’s frame, bounds, or center properties, allowing you to create a static, frame-based layout within Auto Layout.

>Note that the autoresizing mask constraints fully specify the view’s size and position; therefore, you cannot add additional constraints to modify this size or position without introducing conflicts. If you want to use Auto Layout to dynamically calculate the size and position of your view, you must set this property to false, and then provide a non ambiguous, nonconflicting set of constraints for the view.

>By default, the property is set to true for any view you programmatically create. If you add views in Interface Builder, the system automatically sets this property to false.

So, in an AutoLayout-enabled view (what you get from using Interface Builder commonly), every view is expected to have constraints to define its size and position. So, as a convenience, if we ever want to manually layout a specific view in a such AutoLayout-enabled view hierarchy, having `translatesAutoresizingMaskIntoConstraints` set to `true` (the default) results in translating our changes to the `frame` (and `bounds` and `center`) property into constraints. 

So, this is why we set it to `false` if we are adding constraints to a view in code, to avoid ending up with conflicting constraints. But when do we need to set it to `true`? As mentioned in the third paragraph from the docs excerpt above, Interface Builder sets this property to false automatically **because it expects us to add constraints**. 

### When to set `translatesAutoresizingMaskIntoConstraints` to `true`?

So, if you, for any reason, have to define your view in Interface Builder, but also want to manually layout it in code, you'll need to set `translatesAutoresizingMaskIntoConstraints` to `true` before layout.



