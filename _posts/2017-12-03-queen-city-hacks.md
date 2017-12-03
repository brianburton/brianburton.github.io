---
title:  "Queen City Hacks Talk - New Javimmutable Maps"
date:   2017-12-03 13:00:00 -0300
categories: queencityhacks
---
## Inspirational images

- [Boromir Meme](http://vitiy.info/wp-content/uploads/2015/06/immutability.png)
- [Word of the day](https://i.pinimg.com/originals/28/e6/61/28e66144939c77ddb63b6fe528fd34a2.jpg)
- [One more time](http://jr0cket.co.uk/slides/images/mutable-state-reservior-dogs-say-mutable-state-one-more-time.png)


## Illustrations of Some Technical Concepts
- [hamt sharing](http://images.slideplayer.com/28/9400728/slides/slide_21.jpg)
- [compressed trie](https://image.slidesharecdn.com/harshitagarwal11100en006semminarppt-150510121220-lva1-app6892/95/suffix-tree-and-suffix-array-8-638.jpg?cb=1431260294)
- [compressed trie 2](http://img.blog.csdn.net/20151111171557146?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)
- [2 bit hamt path](http://moaazsidat.com/assets/react-immutable/02-BitmappedArrayTrie.gif)
- [compressed trie](http://www.csie.ntnu.edu.tw/~u91029/Trie6.png)
- [rrb trees](https://infoscience.epfl.ch/record/169879/files/RMTrees.pdf)
- [optimizing hamt](https://michael.steindorfer.name/publications/oopsla15.pdf)


## JImmutableHashMap changes

- After last month's talk I decided to revisit the HAMT implementation
    - [rewrite the maps](https://cdn.meme.am/instances/250x250/67313034/rewrite-all-the-maps.jpg)
    - [yeah...](https://memegenerator.net/img/instances/500x/62231893/yeah-if-you-could-just-rewrite-all-that-code-asap-thatd-be-greaaaaat.jpg)
- replaced the HAMT hash map implementation this month
    - different ways to break down integer into 5 bit parts
    - old implementation used bits in order from highest to lowest
        - `index = (hashCode >>> shift) & 0x1f`
        - `shift` is 30 at root, drops by 5 at each lower level
        - leaves have shift `-5` as placeholder
    - new implementation just takes lowest 5 bits
        - `index = hashCode & 0x1f`
        - `remainder = hashCode >>> 5`
        - when making recursive call to child pass `remainder` to child for `hashCode`
    - tradeoffs
        - search depth
            - old way all values stored in leaves
            - new way values stored within branches at all levels
                - if `hashCode == 0` at any level store value in that branch node
                - provides shorter average search depth
        - iteration
            - old way allows iteration in hashCode order using depth-first search
            - new way doesn't allow sensible iteration order
                - lowest order bits control initial order at top of tree
        - complexity
            - old way has to store `shift` value in every branch node
                - extra storage space
                - possible source of error
                - allows index of any leaf to be reconstructed
            - new way
                - no need to track shifts or indexes inside nodes
                - lower storage requirements
                - simpler code
    - performance
        - both are fast but new way is ~20% faster
- removed hash specific code from old implementation and kept it for use with JImmutableArray

### Old Code

````
    @Override
    public T getValueOr(int shift,
                        int index,
                        T defaultValue)
    {
        assert this.shift == shift;
        final int bit = 1 << ((index >>> shift) & 0x1f);
        final int bitmask = this.bitmask;
        if ((bitmask & bit) == 0) {
            return defaultValue;
        } else {
            final int childIndex = realIndex(bitmask, bit);
            return entries[childIndex].getValueOr(shift - 5, index, defaultValue);
        }
    }

    @Override
    public TrieNode<T> assign(int shift,
                              int index,
                              T value,
                              MutableDelta sizeDelta)
    {
        assert this.shift == shift;
        final int bit = 1 << ((index >>> shift) & 0x1f);
        final int bitmask = this.bitmask;
        final int childIndex = realIndex(bitmask, bit);
        final TrieNode<T>[] entries = this.entries;
        if ((bitmask & bit) == 0) {
            final TrieNode<T> newChild = LeafTrieNode.of(index, value);
            sizeDelta.add(1);
            return selectNodeForInsertResult(shift, bit, bitmask, childIndex, entries, newChild);
        } else {
            final TrieNode<T> child = entries[childIndex];
            final TrieNode<T> newChild = child.assign(shift - 5, index, value, sizeDelta);
            return selectNodeForUpdateResult(shift, bitmask, childIndex, entries, child, newChild);
        }
    }

    private static int realIndex(int bitmask,
                                 int bit)
    {
        return Integer.bitCount(bitmask & (bit - 1));
    }
````


### New Code

````
    public T getValueOr(int key,
                        T defaultValue)
    {
        if (key == 0) {
            return filled ? value : defaultValue;
        }
        final int index = key & MASK;
        final int remainder = key >>> SHIFT;
        final int bit = 1 << index;
        if ((bitmask & bit) == 0) {
            return defaultValue;
        } else {
            final int childIndex = realIndex(bitmask, bit);
            return children[childIndex].getValueOr(remainder, defaultValue);
        }
    }

    @Nonnull
    public HamtNode<T> assign(int key,
                              @Nullable T value,
                              @Nonnull MutableDelta sizeDelta)
    {
        final HamtNode<T>[] children = this.children;
        final int bitmask = this.bitmask;
        if (key == 0) {
            if (filled) {
                if (this.value == value) {
                    return this;
                } else {
                    return new HamtNode<>(bitmask, true, value, children);
                }
            } else {
                sizeDelta.add(1);
                return new HamtNode<>(bitmask, true, value, children);
            }
        }
        final int index = key & MASK;
        final int remainder = key >>> SHIFT;
        final int bit = 1 << index;
        final int childIndex = realIndex(bitmask, bit);
        if ((bitmask & bit) == 0) {
            final HamtNode<T> newChild = empty().assign(remainder, value, sizeDelta);
            final HamtNode<T>[] newChildren = ArrayHelper.insert(this, children, childIndex, newChild);
            return new HamtNode<>(bitmask | bit, filled, this.value, newChildren);
        } else {
            final HamtNode<T> child = children[childIndex];
            final HamtNode<T> newChild = child.assign(remainder, value, sizeDelta);
            if (newChild == child) {
                return this;
            } else {
                final HamtNode<T>[] newChildren = ArrayHelper.assign(children, childIndex, newChild);
                return new HamtNode<>(bitmask, filled, this.value, newChildren);
            }
        }
    }
````

## JImmutableTreeMap changes

- replaced the 2-3 tree map implementation this month
    - btree uses 16-32 children per node
        - uses binary search to pick between children
        - much faster than the 2-3 tree
    - implementation is actually simpler since it uses arrays for children
        - old 2-3 tree had two node types (2-node and 3-node) which used fields rather than arrays for children
        - since fields were used separate if branches handled changes for each child
        - duplicated code in each if branch

## Stream collector methods

- added method to every collection type to return a collector
- makes it easy to collect output from any java Stream into any of the collections
- if collection is non-empty the values from Stream are added after existing values

````
    JImmutableList<Integer> x = Arrays.asList(0, 1, 2, 3, 4)
        .stream()
        .map(i -> 2 << i)
        .collect(JImmutables.list().listCollector());
    assertEquals(JImmutables.list(1, 2, 4, 8, 16), x);
    
    JImmutableMap<Integer, Integer> myMap = JImmutables.map().assign(18, 77);
    x.stream()
        .map(i -> MapEntry.of(i, -i))
        .collect(myMap.mapCollector());
    assertEquals(6, myMap.size());
    assertEquals(77, myMap.get(18));
    assertEquals(-16, myMap.get(16));
````



