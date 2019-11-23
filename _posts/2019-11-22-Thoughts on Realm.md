---
layout: post
title:  "Thoughts on Realm"
date:   2019-11-23 13:16:00 +0200
---

Realm is a popular persistence solution, and I really like it a lot.
Because of its ease of use, many - including me - prefer it over Core Data.
However, it has some surprises and limitations.
Here I discuss some of them.

## When Laziness Backfires
Laziness may be the defining feature of Realm.
Performance-wise, Realm saves a lot of execution time by following a lazy approach in retrieving data.
That is, queries are not really evaluated unless the results are accessed.
And data (objects and properties) are not loaded ino memory unless accessed, too.
Consider the following code (from their docs):

```swift
let dogs = realm.objects(Dog.self) // retrieves all Dogs from the default Realm
```
That line alone basically does nothing.
Fetching is done when we start to do something with that result; like accessing the `count` property, or accessing an item at index.

```swift
let dogs = realm.objects(Dog.self) // retrieves all Dogs from the default Realm
print(dogs.count) // Here it begins to really evaluate the query.
```

So, requesting `dogs.count` evaluates the query. But since no data is really fetched, this executes fast.

Realm optimizes for this pattern of use. 
Actually Realm bets on queries being simple, and data being already properly organized and indexed. Consider the following code:

```swift
var tanDogs = realm.objects(Dog.self).filter("color = 'tan' AND name BEGINSWITH 'B'")
```

This is a more complex query. However, it still will be fast whenever we deal with it.
I didn't try a similar query on a very large dataset to confirm speed, but you can figure out how
Realm would optimize retrieving data by following a cursor-like approach, since the expected results will
probably be enumerated (e.g. used in a loop). 
So, it doesn't have to scan all the data to start giving you something to work with.
However, consider the following query:

```swift
let sortedDogs = realm.objects(Dog.self).filter("color = 'tan' AND name BEGINSWITH 'B'").sorted(byKeyPath: "name")
```

Here we use the same query from above except we ask Realm to fetch sorted by name.
Depending on the size of data, we can notice a significant slow down.
The culprit is the sorting phase. The reason is sorting requires Realm to scan the whole data in advance to
start giving you something to work with. So, the full cost of the filtering part will be paid.

Now we're starting to make good sense of how Realm works in regard to data retrieval.
The question is, how this can hurt us?

Notice we use a synchronous API when using Realm. We don't bothering we asynchrony, callbacks, 
threads, etc. We get data in a plain simple way.
And since everything goes as planned, we don't block the UI thread, and everything goes smooth...until we face such query.
Then, problems start, as the docs don't prepare you for such situations (remember, those are "rare" situations). For example, developers will probably start to "fix" this using classic remedies, such as
offloading work to the background. But that only reveals other Realm limitations, like instances not being passable across threads. A very popular issue among Realm users.

Thankfully, helpful strategies are compiled [here by JP Simard](https://github.com/realm/realm-cocoa/issues/4886#issuecomment-296319210). Which can summarized as:

1. Use the async [notification API](https://realm.io/docs/swift/latest/#notifications).
2. Keep the database and objects small.
3. Think carefully about how data should enter the database in first place.

While all are useful advices to follow (even when not using Realm), notice how we then get far from Realm's
marketed simplicity. So, to utilize Realm best, don't think of it as an object-oriented alternative to a [RDBMS](https://en.wikipedia.org/wiki/Relational_database). Think of it as persistence companion to viewing your data in lists. That is, save your data in a way it is as ready to be fetched as-is as possible without much querying and post-processing.

## Invasive Types

Another inconvenience with Realm is that you can't harness its performance offers unless you make 
your code tightly coupled to their types.

Recall that code from above:

```swift
var tanDogs = realm.objects(Dog.self).filter("color = 'tan' AND name BEGINSWITH 'B'")
```

We now know it will run fast as long as we don't force Realm to do the full scan beforehand.
We saw how could we force it by requesting a sort. However, you can force it too by converting the `Results` 
object to a Swift array, as follows:

```swift
var tanDogs = realm.objects(Dog.self).filter("color = 'tan' AND name BEGINSWITH 'B'").map({ $0 })
```
So, you can see how this will affect your design, especially if you're keen on isolating your database 
implementation from the rest of your project.


## Conclusion

Notice some of this applies to other 

At last, if something sounds too good to be true, it probably is.
This is not a rant against Realm (maybe the docs 😅), it's a brilliant solution made by brilliant engineers.
However, we developers should deal with any tool with a grain of salt, and understand its limits before its capabilities.
