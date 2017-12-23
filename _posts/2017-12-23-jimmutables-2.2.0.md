---
title:  "Javaimmutable Collections 2.2.0"
date:   2017-12-23 09:48:00 -0500
categories: javimmutable
---
## 2.2.0 Released!

The latest Java 8 compatible version of Javimmutable Collections has been released!  The [github releases page](https://github.com/brianburton/java-immutable-collections/releases) describes the new features but I'd like to cover some of them in more detail here.

### Better, faster sorted maps

The old `JImmutables.sortedMap()` implementation worked well and had decent performance.  There were a number of aspects of the implementation that I was unhappy with though.  In 2.2.0 the 2-3 tree implementation has been replaced by a b-tree to resolve these issues.

#### Code complexity

The 2-3 tree code was full of repeated code sequences that were identical but referenced different fields in the node.  For example modifying a value down the left branch of a 2-node was very similar to modifying the value down the right branch in a 2-node but since they referenced different fields by name (rather than simple array elements using an index) code had to be copy/pasted and field names changed.

In contrast the b-tree implementation uses arrays in the branch nodes so operations on any branch can use the same code with just an array index being different.  The code simplicity makes the map easier to understand and maintain.

**Sample Code From 2-3 Tree**

````
    @Override
    UpdateResult<K, V> assignImpl(Comparator<K> comparator,
                                  K key,
                                  V value)
    {
        if (comparator.compare(key, leftMaxKey) <= 0) {
            UpdateResult<K, V> result = left.assignImpl(comparator, key, value);
            switch (result.type) {
            case UNCHANGED:
                return result;

            case INPLACE:
                return UpdateResult.createInPlace(new ThreeNode<>(result.newNode,
                                                                  middle,
                                                                  right,
                                                                  result.newNode.getMaxKey(),
                                                                  middleMaxKey,
                                                                  rightMaxKey),
                                                  result.sizeDelta
                );
            case SPLIT:
                return UpdateResult.createSplit(result.createTwoNode(),
                                                new TwoNode<>(middle,
                                                              right,
                                                              middleMaxKey,
                                                              rightMaxKey),
                                                result.sizeDelta
                );
            }
        } else if (comparator.compare(key, middleMaxKey) <= 0) {
            UpdateResult<K, V> result = middle.assignImpl(comparator, key, value);
            switch (result.type) {
            case UNCHANGED:
                return result;

            case INPLACE:
                return UpdateResult.createInPlace(new ThreeNode<>(left,
                                                                  result.newNode,
                                                                  right,
                                                                  leftMaxKey,
                                                                  result.newNode.getMaxKey(),
                                                                  rightMaxKey),
                                                  result.sizeDelta
                );
            case SPLIT:
                return UpdateResult.createSplit(new TwoNode<>(left,
                                                              result.newNode,
                                                              leftMaxKey,
                                                              result.newNode.getMaxKey()),
                                                new TwoNode<>(result.extraNode,
                                                              right,
                                                              result.extraNode.getMaxKey(),
                                                              rightMaxKey),
                                                result.sizeDelta
                );
            }
        } else {
            UpdateResult<K, V> result = right.assignImpl(comparator, key, value);
            switch (result.type) {
            case UNCHANGED:
                return result;

            case INPLACE:
                return UpdateResult.createInPlace(new ThreeNode<>(left,
                                                                  middle,
                                                                  result.newNode,
                                                                  leftMaxKey,
                                                                  middleMaxKey,
                                                                  result.newNode.getMaxKey()),
                                                  result.sizeDelta
                );

            case SPLIT:
                return UpdateResult.createSplit(new TwoNode<>(left,
                                                              middle,
                                                              leftMaxKey,
                                                              middleMaxKey),
                                                result.createTwoNode(),
                                                result.sizeDelta
                );
            }
        }
        throw new RuntimeException();
    }

````

**Greatly Simplified Code From B-Tree**

````
    @Nonnull
    @Override
    public UpdateResult<K, V> assign(@Nonnull Comparator<K> comparator,
                                     @Nonnull K key,
                                     V value)
    {
        final Node<K, V>[] children = this.children;
        final int index = findChildIndex(comparator, key, children, 0);
        final UpdateResult<K, V> childResult = children[index].assign(comparator, key, value);
        switch (childResult.type) {
        case UNCHANGED:
            return childResult;

        case INPLACE: {
            final Node<K, V>[] newChildren = ArrayHelper.assign(children, index, childResult.newNode);
            return UpdateResult.createInPlace(new BranchNode<>(newChildren), childResult.sizeDelta);
        }

        case SPLIT: {
            final Node<K, V>[] newChildren = ArrayHelper.assignInsert(this, children, index, childResult.newNode, childResult.extraNode);
            final int newChildCount = newChildren.length;
            if (newChildCount <= MAX_CHILDREN) {
                return UpdateResult.createInPlace(new BranchNode<>(newChildren), childResult.sizeDelta);
            } else {
                final Node<K, V> newChild1 = new BranchNode<>(ArrayHelper.subArray(this, newChildren, 0, MIN_CHILDREN));
                final Node<K, V> newChild2 = new BranchNode<>(ArrayHelper.subArray(this, newChildren, MIN_CHILDREN, newChildCount));
                return UpdateResult.createSplit(newChild1, newChild2, childResult.sizeDelta);
            }
        }

        default:
            throw new IllegalStateException("unknown UpdateResult.Type value");
        }
    }
````

#### Spliterators

One of the operations required to implement parallel Streams in java is the ability to split a Spliterator efficiently into two Spliterators of roughly equal size.  Currently Javimmutable does this at the root node of the tree.  This means that a 2-3 tree can split at most 3 times while a b-tree could potentially split 32 times.  Of course the width of the root of the b-tree varies with the number of elements in the tree.  Small trees will still not split well but parallel processing isn't all that important for small trees.

#### Performance

For a balanced tree the performance is generally dependent on how deep the tree can become.  The 2-3 tree has at most 3 branches per level of the tree.  In contrast the b-tree can have 16 to 32 branches per level.  This means the b-tree will have significantly lower depth on average (a 1 million node tree would have approx 5 depth at 16 branches per level vs 20 at 2 branches per level).

Benchmarks showed the new b-tree performing at least 10% faster on average than the 2-3 tree.

Implementation|Map Size|Milliseconds
---|--:|--:
java mutable TreeMap|166k|324
immutable 2-3 tree|166k|537
immutable b-tree|166k|474

As the benchmarks show both implementations have comparable performance to the mutable java.util.TreeMap class.  The new implementation narrows this gap slightly.

### Better, faster hash maps

#### Background

The key to decent performance in an immutable hash map is the Hash Array Mapped Trie.  As with most things in software there are different ways to implement these.  The original implementation in Javimmutable Collections grew out of the intention to offer both a hash map and a sparse array.  Since the sparse array **is** an array mapped trie I chose to reuse the sparse array to implement the hash map.

This was great from a code reuse standpoint.  However it also had some unintended consequences.

- The sparse array has to be able to iterate over elements in signed integer order.  Hash maps don't really care about iteration order at all.
- Hash maps have to deal with hash code collisions in keys.  Sparse arrays don't care about keys at all (only indexes).

These two contrasting requirements wound up making both the sparse array and hash map implementations somewhat cumbersome and sub-optimal.  They worked fine and performed well but there was room for improvement.

The 2.2.0 release decouples the implementations of sparse arrays and hash maps.  Each of them have their own array mapped trie implementations with only the features they need to meet their own requirements.

#### Faster, Lighter Hash Maps

To provide signed integer iteration order the sparse array has to store all values at the leaves of the trie.  No values can be stored inside the nodes.  This leads to the trie being somewhat deeper than it has to be if you don't care about iteration order.

The new hash map implementation stores values inside of branch nodes as well as in leaves.  Values stored in branches are reachable at a shallower depth so accessing them is generally faster.  Also since the new implementation doesn't care about iteration order it's able to store less book keeping information per node and the logic to operate on nodes winds up being simpler.

The end result of these savings is improved performance, lower memory footprint, and simpler code.

Implementation|Map Size|Milliseconds
---|--:|--:
java mutable HashMap|150k|59
old immutable hash map|150k|118
new immutable hash map|150k|110


### Stream collector methodsPrevious 2.x versions added support for java 8 streams.  These methods made it easy to create streams from any collection but provided only limited support for creating Collectors to build collections from streams.  This support was added via static methods in the JImmutableCollectors class.

Version 2.2.0 improves Collector support by adding methods to every collection to create a new Collector object based on the collection itself.  If the collection already contains values these will also appear in the new collection.  This makes it easy to build and update collections using multiple streams.
````    JImmutableList<Integer> x = Arrays.asList(0, 1, 2, 3, 4)        .stream()        .map(i -> 2 << i)        .collect(JImmutables.list().listCollector());    assertEquals(JImmutables.list(1, 2, 4, 8, 16), x);    JImmutableMap<Integer, Integer> myMap = JImmutables.map().assign(18, 77);    x.stream()        .map(i -> MapEntry.of(i, -i))        .collect(myMap.mapCollector());    assertEquals(6, myMap.size());    assertEquals(77, myMap.get(18));    assertEquals(-16, myMap.get(16));````