---
layout: post
title:  "Experimenting With targetContentOffset: Part 1: Uneven Pagination"
date:   2019-01-19 15:28:00 +0200
---

## Introduction

There are at least three ways of paginating content in iOS. Namely, via `UIScrollView`, `UIPageViewController`, and `UICollectionView`.

For simplicity, I'll consider only horizontal pagination from now on in this post.

`UIPageViewController` paginates its contents by setting its `transitionStyle` property to `.scroll`. `UIScrollView` and `UICollectionView` paginate their content by setting their `isPagingEnabled` property to `true`.

It's worth noting that all these solutions are essentially built upon `UIScrollView`. `UIPageViewController` uses a special `UIScrollView` subclass (private API I think) called `_UIQueuingScrollView`. `UICollectionView` is a `UIScrollView` subclass.

## Limitations
### 1. Page Size is Fixed

A common trait among the previous solutions is the fixed page size. That is, the amount by which content is paged is always equal to the scrollView frame. If you want to have a page size different from the visible "frame", you have to seek workarounds; e.g. this clever solutions [[1](http://khanlou.com/2013/04/changing-the-size-of-a-paging-scroll-view/), [2]((http://khanlou.com/2013/04/paging-a-overflowing-collection-view/))] by Khanlou.

### 2. Uneven Page Size

Another limitation is if you want more than a page size in a single flow. For example, when your flow can be considered a series of pairs, where each pair of pages are separated by a constant spacing, while each item of each pair is separated by a different amount of spacing. I think this is impossible to work around with the above solutions.

![uneven1]({{site.url}}/assets/uneven1.png)

![uneven2]({{site.url}}/assets/uneven2.png)

One can think of having more than one level of pagination to fix this. For example:

1. A `UIPageViewController` for the pairs, while each pair is a `UIpageViewController` itself.
2. A `UICollectionView` for the pairs, while each pair is a `UICollectionView` itself.
3. Similar thing with raw `UIScrollView`.

However, these solutions have problems.

1. For the nested `UIPageViewController` it's so easy to swipe an entire pair while not noticing. This is because the outer and the inner `UIPageViewController`s have contentSize greater than the visible frame (since `UIPageViewController` always loads 3 pages if possible (left, center, right)). So, any pan gesture can both affect any of them.

2. Similar thing can happen too with nested `UICollectionView`s. However, it can be worked around by disabling prefetching on the outer `UICollectionView`. This way, the outer `UICollectionView` only loads one cell (pair cell), while the pair cell can load its full content; so the pan gesture would work fine with inner `UICollectionView` as expected. However, on fast scrolling, this seems to not work; and pairs are again skipped.

## `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)` has something to say


`UIScrollViewDelegate` has this interesting method that is called when the user ends dragging. It reports the velocity by which the user did their swipe, and (which is our focus) passes the expected content offset at which the scrollView would stop! So clever! And there's more to it. It's possible to change that expected offset so the scrollView smoothly stops at a desired position!

So, knowing this, we can "snap" the decelerating scrollView to a position of our choice, so there is a chance to solve our uneven page size problem.

### Idea

(Using a `UICollectionView` of pairs, where each cell is a pair of `UIView` subclass)

Given an item width equal to the visible frame. Each pair of items are separated by 50 pts of space on each side. We can:

1. We can partition of content into a series of evenly sized pairs (including spacing). That is, each pair width = item width * 2 + spacing (25 pts at each side).

2. When `scrollViewWillEndDragging` is called, we can inspect the `targetContentOffset` and see at what index of pairs that offset value should correspond. Such index can be achieved by dividing (integer division) the value of the `targetContentOffset` by the pair width.

3. Note that `targetContentOffset` always points to the leftmost of the screen. This causes a bias to the left side of scrolling, so that way, integer division would be inclined to get lesser indices; 1 is more likely to come than 2, 2 is more likely to come than 3, and so on... One way to overcome this is to offset the `targetContentOffset` a little to balance this bias; making it points to the middle of the screen rather than it leftmost edge. To achieve this, just add half of the visible frame width to the `targetContentOffset` before integer division.

4. Now we have a correct index of a pair. We only have to decide which part of the pair we want to snap to. So, the sizes of each part of the pair should be known to us. That way we can decide which part is close to the adjusted `targetContentOffset` calculated above (adjusted to the middle of visible frame). And that's it. Now finally alter the value `targetContentOffset` to achieve our desired effect, e.g.: `targetContentOffset.pointee.x = rightItemX`.

Notes:

1. We don't use `isPagingEnabled` here; we use normal scrolling. The default decelaration rate may be too slow; so setting the `UICollectionView`'s `decelerationRate` to `.fast` should do it.

2. We have to inset the `UICollectionView` by half the spacing at each side to achieve contentSize multiple to that of pair width.

3. Large swipes may cause jumping over a page. This avoidable by clamping the amount by which the `targetContentOffset` changes. It may also appear as a feature not a defect. 😄

## Conclusion

Here is a [demo in Swift](https://github.com/ahmedk92/UnevenPagination) that implements what's above.

Although the solution presented here may not be perfect, it's the best I could come up with. It's a tricky problem that I hadn't find a complete solution for it so far.

For questions, and suggestions please contact me via Twitter, or submit a pull request on the linked Github demo.

Thanks for reading!





