---
layout: post
title:  "Realm: Lessons Learned"
date:   2019-11-22 01:20:00 +0200
---

I really like [Realm](https://realm.io/docs/swift/latest). Its strongest selling points are ease of use, speed, and memory-efficiency. However, Realm can be tricky and a source of surprising problems. Here I discuss some of those.

I'm going to split the points into three main points: 
1. [Problems specific to Realm](#problems-specific-to-realm).
    
    1.1. [Misleading Laziness](#misleading-laziness)

    1.2. [Invasive Types](#invasive-types)
2. [Problems with Live-Object-Based persistence frameworks](#problems-with-live-object-based-persistence-frameworks).
3. [Alternatives](#alternatives).

## Problems specific to Realm

### Misleading Laziness
Maybe the first thing that gets you sold when considering Realm is little code to query objects like this:

```swift
let dogs = realm.objects(Dog.self) // retrieves all Dogs from the default Realm
```
Little to no code. Absolutely no threads involved. And it will be fast.

However, this design decision hides a lot of details that can hurt us later.
First, what this line does? It just returns a `Results` object which is like a list that holds nothing but references to objects on disk.
So, since it doesn't retrieve any data, it returns fast. Not just that, but that line alone actually does nothing if we don't do anything with the returned results like accessing an object at index, or the `count` property.

You may ask, what's the problem with this?

The problem with such approach is that this marketed synchronous API doesn't scale with a more complex query. That is, if the data is large enough, a sorting query or string-scanning query can result in a really slow performance that blocks the UI thread, like [this](https://github.com/realm/realm-cocoa/issues/4886).
The alternative async solution proposed by jpsim is much less convenient that the sync one, as it needs us to deal with [notifications](https://realm.io/docs/swift/latest/#notifications).

So, you have to think how large and complex your data is, and think about your queries. Which is not a surprise when dealing with databases.
However, Realm's marketing and docs promote a magical fast and easy solution that refrains you from that mindset, especially beginners. This is a good reminder that if something seems too good to be true, it probably is too good to be true.

### Invasive Types



## Problems with Live-Object-Based persistence frameworks
## Alternatives