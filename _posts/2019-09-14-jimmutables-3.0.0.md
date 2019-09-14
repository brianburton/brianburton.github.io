---
title:  "Javaimmutable Collections 3.0.0"
date:   2019-09-14 13:43:00 -0500
categories: javimmutable
---
## 3.0.0 Released!

The latest version of Javimmutable Collections has been released!  The [github releases page](https://github.com/brianburton/java-immutable-collections/releases) describes the new features but I'd like to cover some of them in more detail here.

### Do we really need JImmutableRandomAccessList?

As an experiment I decided to rewrite the JImmutableTreeList as a modified AVL tree.  The new implementation used the same invariant (depth of each node's children cannot differ by more than one) as an AVL tree and stored child values in leaves at the bottom of the tree.  The leaves themselves contain an array of values that can grow or shrink from 2 to 64 values.  There are special case implementations for zero or one value leaves.

It turns out that the rotations needed to keep the trees in balance work much better in an immutable context than I had originally expected.  They are still a bit complicated to work out but I managed to do so over a piece of note paper with some tree diagrams.

The resulting list was *much* faster than the B-Tree implementation.  The reason for this is simply that the binary tree nodes can very efficiently determine which child to pass a call down to.  In contrast the B-Tree had to scan an array of children linearly to find the right child to call.

As a bonus the new implementation uses less RAM since leaves could hold many more children than was true in B-Trees.  And the code was less complicated overall!

After seeing how much faster the balanced tree was than the B-Tree I decided to revisit the case for having separate interfaces for lists that support insertion/deletion only at the ends (JImmutableList) or anywhere in the list (JImmutableRandomAccessList).  The JImmutableArrayList was still faster than the tree list (about 40% faster in my benchmarks).  That didn't seem like a big enough margin to keep the distinction around though.

Removing JImmutableRandomAccessList was a big change to the library.  It could definitely produce compiler errors for old code where people used the old interface.  I decided it was worth the backward compatibility hit though to clean up the library code.  I was able to remove quite a few classes in the process.  If you used JImmutableRandomAccessList before you should be able to do a quick search/replace to change JImmutableRandomAccessList to JImmutableList and have your code compiling again in little time.

### Appending and slicing lists

Slimming down to a single list interface also allowed me to incorporate  append functionality right into JImmutableList.  This allowed methods like `insertAll()` to work in O(log2(n)) time instead of O(n) time when the Iterable being inserted is another JImmutableList.

It turns out that once you have an efficient append operation in a list it becomes trivial to write an efficient truncate operation.  The new list can truncate efficiently from either end of the list.  This made methods like `prefix()`, `middle()`, and `suffix()` fast and easy to write.  As always the lists returned by these methods reuse most of their contents from the original tree.  I also added the `slice()` method as an experiment to see if negative indexing a.la. Python would be useful.


### Better sorted maps

Since I was on a balanced binary tree kick I decided to tackle a rewrite of the sorted maps as well.  Previously these used B-Trees that were a great improvement over the prior 2-3 tree implementation.

The new binary tree implementation has a number of advantages over the B-Tree though.

Every node in the tree contains a value.  This means that the number of nodes in the tree is exactly equal to the number of keys in the map.  This results in an overall memory savings.  This can be particularly helpful when using the binary tree nodes within a hash map for collision handling.

Code complexity is reduced somewhat with the binary tree.  You have the extra complexity of the tree rotations but this is made up for by the elimination of the merge and split functionality of the B-Tree.  Overall its probably a slight win for the binary tree.

### Do we really need Cursors?

I loved the Cursor concept when I first wrote the libary.  They were immutable and cool.  You could use them to easily implement look ahead functionality that was impossible with Iterators.

Unfortunately they had some major down sides:

- You create a new Cursor object for every element you visit.  This can create a lot of little objects to be removed by the GC.  This isn't a huge problem since they are short lived (they might even be detectable by the JIT using escape analysis) but why introduce the memory pressure when its not necessary?
- Java uses Iterators everywhere and has terrific support for using them in loops and Streams.  Why have our own, incompatible, way of iterating?
- Cursors can't be used with Streams.

So I decided that since this was a compatibility breaking release anyway I may as well remove the Cursors.  That turned out to result in big reduction in code.  Most classes had Cursor code that duplicated the Iterator code.  There were Cursor utility classes, specialized Cursor implementation classes, etc.  All in all removing them was a big win from a maintainability standpoint.

### How far can you split?

We've had Spliterator support in all collections ever since 2.0.  However the granularity of the Spliterators has been pretty dismal.  Each data structure had its own quirks regarding how many times you could split.  Some didn't support more than a couple of splits.

Since I was doing away with Cursors I decided that I should optimize our support for Streams.  So GenericIterator was born!  GenericIterator is able to split into generally equal sized Iterators all the way down to a defined limit in size.  I chose 32 as the limit but that's arbitrary.  I just guessed that this would be a point where the overhead of using another thread wouldn't be worthwhile any more.

The GenericIterator relies on the ability of the various collections to be able to navigate in roughly O(log(n)) time to any arbitrary value in its tree/trie.  Since this doesn't happen until iteration actually starts Java's Stream implementation class is able to split as much as it wants and then this navigation happens quickly at the start of the iteration in each thread.

Supporting this required changes to every collection.  However the changes were worthwhile and now when you want to do parallel processing of large collections using Streams you should get excellent distribution of the work over a large number of threads without problem.

Having every collection use the same Iterator implementation removed  complexity from the collections themselves.  The GenericIterator uses a stack of state objects to be able to move around inside the structure easily.  The stack's height is never more than the height of the structure it's navigating.  There are only a few types of state objects and GenericIterator provides well tested implementations of all of them for use by the collections.  This keeps the iteration logic in one class.

This might be the most satisfying change in the release since I felt the Spliterator support in prior releases wasn't nearly as good as I wanted it to be.

### Want to build a map?  How about a set?

Previous releases had Builder implementations for lists.  Only for lists.  Those Builders were nice and fast.  If they are good for a list why not for a map or a set?

Since sets are built on maps writing a Builder for the hash and tree maps makes writing a builder for sets trivial.  But writing a builder for a map or array might not be so trivial.

Sorted maps are based on a binary tree.  You can build them efficiently using a list of values.  But of course those values have to be sorted.  And you have to keep in mind that users might call `add(k,v)` multiple times with different values for the same key.  So I implemented the building using a standard TreeMap.  I wasn't thrilled about that but I think it was the right call.  I could have made a mutable binary tree implementation for the builder but that didn't seem worth the effort and maintenance.

Hash maps are strange beasts.  The hash array mapped trie is an awesome data structure.  It's also pretty strange though.  There really isn't an efficient way to build one efficiently from a list of values.  So I wound up writing a mutable version of the trie with no array compression and only supporting addition to the trie.  This was complicated but it worked out nicely.  Once you have a mutable trie it's fast and easy to build up a compact immutable version from the mutable nodes.

### Final thoughts

This release was fun to put together and provides many useful enhancements.  I'm proud of it and looking forward to using it in other projects.  I hope others find the changes useful too!
