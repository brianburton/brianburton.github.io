---
title:  "Javaimmutable Collections 3.2.1"
date:   2021-01-01 12:00:00 -0400
categories: javimmutable
---
## 3.2.1 Released

The latest version of Javimmutable Collections has been released!  The [github releases page](https://github.com/brianburton/java-immutable-collections/releases) describes the new features but I'd like to cover some of them in more detail here.

Actually this post covers the changes from both 3.2.0 and 3.2.1. 

### Simpler, smaller arrays

JImmutableArray has always been the simplest HAMT implementation since it had no need to support collision maps.  It was still a bit more complicated than I liked.  Plus I'd always preferred the idea of having a single HAMT to serve as a basis for hash sets and maps as well as arrays.  Once 3.1.0 was released I used my enforced non-work time to work on an improved array implementation.

The first improvement was exploiting twos compliment to simplify iteration order handling.  The new implementation takes every user provided index and flips its high-order bit (i.e. 0 becomes 1 and vice versa).  This converts the user space index range of signed integers [-2^31,2^31) to an equivalent unsigned range [0,2^32) (`[` means inclusive, `)` means exclusive).  This makes iteration in user signed index order easy as long as our HAMT can support iteration in internal unsigned index order.

Next I decided to take a trick from the AVL tree implementaton and store values internally within the trie rather than just at the leaves.  This complicates the nodes themselves slighly but has the effect of reducing the number of nodes in the trie.  Fewer nodes means lower memory footprint.

To further reduce the number of nodes in the trie I added height compression.  Whenever a parent node detects that a child node would naturally be multiple levels lower in the trie than itself and there is currently no child higher in the trie than the new one it simply stores the child itself without creating its ancestors.  To see how this works assume you have an empty trie.  Next you insert an element that would naturally appear at level 4 in the trie.  Without height compression the root would now point to a level 1 node that just has a level 2 node as its child.  The level 2 node would just have a level 3 node as its child.  The level 3 node would have our new value node as its child.  Thus we have a trie of height 4 when all we really wanted was to store a single node!

With height compresison level 4 node is still created but it becomes the root of the trie and none of the parents are needed. The trie nodes are smart enough to create or remove intermediate nodes as necessary when values are added to or deleted from the trie.  This keeps the array's trie as small as possible at all times.

All of the logic necessary for a trie could now reside in a single class.  The old class hierarchy for array nodes was removed since it was now redundant.

With this change in place the memory footprint of arrays is roughly half what it was in 3.1.0.

### Even smaller maps and sets

With the smaller array trie in place I was able to take another look at the hash map and set implementations.  Previously they used their own HAMT implementations that understood how to handle collisions and didn't need to care about iteration order.  The new array implementation didn't pay any price for its deterministic iteration order though and all it needed to support collisions was an adaptor to map the values in the array to CollisionSet Nodes.

So I added a set of mapping interfaces to define how map/set values should be converted into array values and vice versa.  For example `ArrayAssignMapper<T,K,V>` knows how to convert a map's key and value into the appropriate value to store in the array trie (i.e. an `ArraySingleValueMapNode<K,V>` or `ArrayMultiValueMapNode<K,V>`).  Then I added methods to the array node to handle these mapped assign, delete, etc operations.  This bulked up the array node code a bit but allowed me to delete the map and set HAMT implementations completely.

Now hash maps and sets use the same trie code as arrays.  Their memory footprints dropped accordingly and a number of classes could be removed.

### Memory Footprint Progress

The table below shows how the memory footprint of the classes has changed over the last few releases.  As you can see the change is significant.  JImmutableHashSet and JImmutableHashMap are now smaller than their `java.util` equivalents.   Even insert order maps are only slightly larger than their mutable equivalent.

Collection | Java Equivalent Size | 3.0.0 size | 3.1.0 size | 3.2.1 size | % of Java Size
---|---|---|---|---|---
JImmutableHashMap | 5846608 | 9254408 | 6032632 | 4512936 | 77%
JImmutableHashSet | 5846624 | 9254424 | 6032632 | 3712936 | 64%
JImmutableTreeMap | 5598000 | 5598008 | 5597992 | 5597992 | 100%
JImmutableTreeSet | 5598032 | 5598032 | 5598016 | 5598016 | 100%
JImmutableInsertOrderMap | 6646600 | 17254472 | 14032680 | 7440512 | 112%
JImmutableTrieArray | n/a | 4540024 | 4540024 | 2112904 | n/a


### Final thoughts

The quest to lower memory footprint has born fruit.  The collections are now leaner.  Our hash maps and sets are smaller than their mutable equivalents.  Even insert order maps are within a whisker of LinkedHashMap's size.  The latter is huge improvement from where we started and is fairly remarkable since we can't use a simple data structure like a doubly-linked list the way the mutable version can.

