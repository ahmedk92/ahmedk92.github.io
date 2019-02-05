---
layout: post
title:  "Experimenting With targetContentOffset: Part 2: Custom Pickers"
date:   2019-02-03 00:12:07 +0200
---

This post should be more fun than its [predecessor]({{ site.baseurl }}{% post_url 2019-01-19-Experimenting With targetContentOffset: Part 1: Uneven Pagination %}). We're going to make a snappy picker control similar to `UIPickerView`, utilizing (of course) `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)`.

I don't intend this to be a tutorial, so this may not be up to your expectations. 😅

I also don't intend this to be a library/resuable view. I like this to be a DIY (Do it yourself) guide in which you learn the concept then customize and create as freely as you want.

> Final code is [here](https://github.com/ahmedk92/MoodPickerDemo)

Enough talk, let's work!

## Demo

![moodpicker]({{site.url}}/assets/moodpicker.gif)

## Idea

As we saw in the previous post, `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)` enables us to control at what point the ongoing scrolling animation should stop. I cannot ***not*** get excited when I think about this.

So, the idea is to "partition" the available content size of the collection view into chunks where each is of the same width which is a multiple of the total content size. Hence, when `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)` gets called, we can check at which partition `targetContentOffset` lies. This gives us the target index of our item.

Now that we have the target index, we can calculate the content offset (point) that corresponds to that index. Multiplying the index by the width of each cell gives us the left-most point of the cell.

```swift
        var targetIndex = targetContentOffset.pointee.x / cellSize.width
```

## Fixing Rounding Bias

I've been talking about calculating an index. An index is an integer value; so how are we rounding that result of dividing `targetContentOffset` by the cell size? To elaborate, the `targetContentOffset` can be something like 199 while the cell size may be 50; so the division operation would result in 3.98. What value are going to choose? 3 or 4?

I don't know the perfect rounding strategy here, but I suggest rounding with respect to the scrolling direction. That is, if we're scrolling right, we round up, and round down if we're scrolling left. We can know the scrolling direction via the `velocity` parameter in `scrollViewWillEndDragging(_:withVelocity:targetContentOffset:)`; left is velocity < 0, right is velocity > 0.

```swift
        targetIndex = velocity.x > 0 ? ceil(targetIndex) : floor(targetIndex)
```

This will run fine in most cases. However, we'll get a not so pleasant result when the velocity is exactly zero. That's if we consider a velocity of zero is either left or right, we can notice that "slow" scrolling is biased towards that direction. What I mean by slow scrolling is when your scrolling gesture is more of a slow pan rather than a quick swipe; something similar to the slide to answer gesture.

One solution to this problem is to add half the width of the cell to the `targetContentOffset` before division. I went with that and works fine with one caveat. When scrolling to the last item, this way of rounding results in an index value greater than the last valid item index by one. I work around this by [clamping](https://en.wikipedia.org/wiki/Clamping_(graphics)) the resulting index.

```swift
        var targetIndex = (targetContentOffset.pointee.x + cellSize.width / 2) / cellSize.width
        targetIndex = velocity.x > 0 ? ceil(targetIndex) : floor(targetIndex)
        targetIndex = targetIndex.clamped(minValue: 0, maxValue: CGFloat(emojis.count - 1))
        targetContentOffset.pointee.x = targetIndex * cellSize.width
        
        index = Int(targetIndex)
```

## Insets

If we implement till this point, we'll find that there are a couple of cells at the left side that we can't center. This is because they're logically at the correct content offset (i.e. The first should be at zero). So, what we need to change here the edge insets of our collection view. We need to inset horizontally with enough amount that the left-most cell is centered (screen-wise), and similarly for the right-most, while maintaining correct content offsets.

The needed inset amount is equal to half the width of the screen minus half the width of the cell.

```swift
    override func viewDidLayoutSubviews() {
        spacing = collectionView.bounds.width / 2 - cellSize.width / 2
    }

     func collectionView(_ collectionView: UICollectionView, layout collectionViewLayout: UICollectionViewLayout, insetForSectionAt section: Int) -> UIEdgeInsets {
        return UIEdgeInsets(top: 0, left: spacing, bottom: 0, right: spacing)
    }
```

## Note

As [Amr Mohammd](https://twitter.com/iAmrMohamed/status/1091955349744177152?s=20) commented, in case of using `UICollectionView` we can avoid the maths done in our `scrollViewWillEndDragging` method by utilizing `UICollectionView.indexPathForItem(at:)`.

```swift
        let point = CGPoint(x: targetContentOffset.pointee.x + scrollView.frame.midX, y: targetContentOffset.pointee.y)
        guard let indexPath = collectionView.indexPathForItem(at: point) else { return }
        index = indexPath.row
        
        targetContentOffset.pointee.x = CGFloat(index) * cellSize.width
```

## Conclusion

Nothing so fancy. I hope you find it useful. Full demo source is [here](https://github.com/ahmedk92/MoodPickerDemo).