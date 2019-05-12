---
layout: post
title:  "Simple Overflowing Paginated UIScrollView"
date:   2019-05-12 10:44:00 +0200
---

So, I was using this beautiful Quran app [Ayah](https://itunes.apple.com/app/apple-store/id706037876?mt=8). I noticed something cool about how it pages its content; there is a visual divider between each two inner pages, and a different one between outer pages. See the below gif for clarity.

![Overflowing Pagination]({{site.url}}/assets/ayah1.gif)

## Preliminary Analysis

You may see such effect being called **"Overflowing Pagination"**. This is not uncommon, and there is good write-ups on how it can be achieved; for example: Soroush's articles [1](http://khanlou.com/2013/04/changing-the-size-of-a-paging-scroll-view/) & [2](http://khanlou.com/2013/04/paging-a-overflowing-collection-view/). Also allow me to plug in an [earlier experimentation of mine]({{ site.baseurl }}{% post_url 2019-01-19-Experimenting With targetContentOffset: Part 1: Uneven Pagination %}) 😁.

## But...I overthought the problem. As usual.

Techniques mentioned above deal with a trickier problem; paging with a page size different than the scrollView's bounds width (the default behavior you get with `isPagingEnabled`). Luckily, to achieve what we saw in the video, we don't need any of this.

If you notice, you'll see that although there are two different looking separator views, they are of the same size. So, this gives us an idea. Instead of having the width of the scroll view being equal to our screen, we increase it by how big we want our separator views to be (with the extra width being evenly distributed over both sides). Moreover, we center the scroll view in which the extra portions are off-screen. And that's it. Here's a [sample code](https://github.com/ahmedk92/SpacedPageViewController), and a demo of it below.

![Overflowing Pagination]({{site.url}}/assets/overflowing.gif)

And as you may already know, as this applies to `UIScrollView`, then it applies to `UIPageViewController` (as in the linked sample) and `UICollectionView` with `isPagingEnabled`.

Thanks. Looking forward for your feedback.