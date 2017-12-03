---
title:  "Javaimmutable Collections 1.9.1"
date:   2017-10-14 12:51:00 -0400
categories: javimmutable
---
## 1.9 Released!

The last planned Java 6 compatible version of Javimmutable Collections has been released!  It's been a long time coming which makes is all the more satisfying to have it out there.  The [github.com releases page](https://github.com/brianburton/java-immutable-collections/releases) describes the new features but I'd like to cover some of them in more detail here.

### Better, faster random access lists

The old `JImmutables.ralist()` implementation worked well and had decent performance but I was never happy with the performance differential between it and `JImmutables.list()`.  Logically you'd expected there to be a price to pay for the extra flexibility to insert and delete elements at any index of the list.  In fact you can see that well in the timing results (from RAListTimingComparison benchmark program that comes with JImmutables) for both java.util.ArrayList and the old JImmutableTreeList.

As the results below demonstrate both collections take longer to complete the benchmarks as the lists becomes larger.  Surprisingly in direct comparison for very large lists the JImmutableTreeList didn't do all that badly.  It may seem strange at first that ArrayList is so slow for larger lists but keep in mind that ArrayList has to shift elements in a large array whenever values are inserted or removed in the middle of the array.  In contrast JImmutableTreeList just had to create a leaf and `log2(n)` branch nodes for each operation.  This advantage only becomes apparent in large lists.

Implementation Class|List Size|Milliseconds
---|--:|--:
ArrayList|1002|57
JImmutableTreeList|1002|**373**
ArrayList|10000|186
JImmutableTreeList|10000|**573**
ArrayList|49998|764
JImmutableTreeList|49998|**754**

Starting in 1.9.1 the JImmutableRandomAccessList implementation uses a b-tree.  While it may seem odd to use a b-tree for an in-memory data structure (aren't those only for databases?!?) it turns out to be a perfect fit.  A 2-3 tree is just a special case of a b-tree after all.    Using a b-tree node with 9-18 values per node decreases the depth of the resulting tree substantially.  Since creating new nodes is the costliest part of creating a modified tree this reduces the amount of work to be done for an insert or delete substantially.

Compare the performance from the previous table that that of the new JImmutableBtreeList introduced in 1.9.1.  

Implementation Class|List Size|Milliseconds
---|--:|--:
ArrayList|1002|57
JImmutableBtreeList|1002|**179**
ArrayList|10000|186
JImmutableBtreeList|10000|**278**
ArrayList|49998|764
JImmutableBtreeList|49998|**394**

Overall performance of the b-tree is nearly double that of the old 2-3 tree.  Also notice that for large lists the new b-tree based immutable list is substantially **faster** than a mutable ArrayList!  That result surprised me the first time I saw it but it's actually logical once you think about it.  The b-tree is significantly shallower than a 2-3 tree and each node in the tree has only 9-18 nodes in its array of values.  Much faster to copy an 17 node array than to shift tens of thousands of values in a huge mutable array.

So 1.9.1 has vastly improved on the JImmutableRandomAccessList performance.  Does that mean we can simply lose the JImmutableList implementation and have one list to rule them all?  That would be nice since it would eliminate the need for two list interfaces and reduce the code in the library.  Unfortunately (and not surprisingly) we can't do that.  The specialized JImmutableArrayList is well optimized for its task (insertion and deletion only at either end) and easily outperforms JImmutableBtreeList in benchmarks (ListTimingComparison).

Implementation Class|List Size|Milliseconds
---|--:|--:
ArrayList|1024|17
JImmutableArrayList|1024|48
JImmutableBtreeList|1024|80
ArrayList|10000|19
JImmutableArrayList|10000|71
JImmutableBtreeList|10000|116
ArrayList|49999|20
JImmutableArrayList|49999|84
JImmutableBtreeList|49999|158

As you can see from the benchmarks the specialized JImmutableArrayList is nearly twice as fast as JImmutableBtreeList at doing inserts and deletes only at the ends of the list.  Also notice that ArrayList shines in this operating mode.  The mutable list can pre-allocate an extra large array with padding at either end to reduce the number of times it needs to shift values around.  This is the best case scenario for ArrayList and that shows in the data.

### Multisets

[Guava](https://github.com/google/guava) contains a mutable class named Multiset.  This interesting collection is similar to a set but keeps track of how many times each value has been added or removed.  Back in the [Smalltalk-80](https://dl.acm.org/citation.cfm?id=273) days there was a class named Bag which did similar things.

The JImmutableMultiset added in 1.9.1 serves the same basic purpose.  You can add and delete items to the multiset and query later to determine how many of each item are in the set.  This can be useful in various situations like counting unique words in a text file, how many times a particular operation has been performed, etc.

In addition to simply providing these counts the JImmutableMultiset also interoperates well with normal JImmutableSet in logical ways.  For example you can perform a union or intersection between multiset and sets (in either direction).  When combining a set with a multiset the latter treats the set as a multiset with only 1 occurrence of each value.  Going the other direction when combining a multiset with a set the latter treats the multiset like an ordinary set.

### Maps of sets

JImmutableListMap is one of the more useful and unique collections in the library.  It makes it easy to keep track of multiple lists by keys.  Sometimes, however, this isn't ideal for a given task since a list can have duplicates added to it.  1.9.1 adds JImmutableSetMap which acts much like JImmutableListMap but associates sets with the keys instead of lists.  Not a huge innovation but it fills a gap in the library and should prove useful in many applications.


