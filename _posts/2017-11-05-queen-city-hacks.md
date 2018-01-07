---
title:  "Queen City Hacks Talk - Javimmutable Overview"
date:   2017-11-05 13:00:00 -0500
categories: queencityhacks
---
## Javimmutable Collections

- **Worlds worst project name**
- [github project](https://github.com/brianburton/java-immutable-collections)

![Banner](https://cdn-images-1.medium.com/max/734/1*mA2NV6aiBt4iVKFGqFmy5g.jpeg)

- Why Immutability?
    - thread safety since race conditions are impossible with immutable data structures
    - safe to pass as parameter values or return from methods
        - return value especially nasty with Collections.unmodifiableMap and friends
    - eliminate need for bad stuff like locking and defensive copying
    - [Good Advice](http://vitiy.info/wp-content/uploads/2015/06/immutability.png)

- Kinds of immutable data structures
    - Pseudo-immutability - Collections.unmodifableList, etc
    - Guava ImmutableList, ImmutableMap, etc
        - defensive copying during creation then unchangeable without another full copy
        - no insert, delete, put etc methods provided since they would be slow
    - Functional/Persistent immutability
        - any given copy of structure never changes but mutated variations can be created easily
        - efficient creation of modified versions of data structure
        - insert, delete, put etc methods are provided that return new version of structure reflecting the change

- Isn't that slow?
    - **NO**!!
    - structure sharing makes it efficient
    - re-use almost entire data structure in every modified variation
    - creating variation requires creating ~ log2(n) or log32(n) new nodes
        - log2(1 million) = 20
        - log32(1 million) = 4
        - so changing a value in a 1 million node 32-way trie only creates 4 new nodes and re-uses all others
    - These images illustrate structure sharing concept (though none of them exactly match how javimmutable does things)
        - [slist](https://image.slidesharecdn.com/functionalprogramminginclojure-101125061148-phpapp01/95/functional-programming-in-clojure-47-728.jpg?cb=1332252249)
        - [tree-1](http://img.blog.csdn.net/20151105061537224)
        - [tree-4](http://arqex.com/wp-content/uploads/2015/02/trees.png)
        - [tree-2](https://voormedia.com/blog/2013/02/creating-immutable-tree-data-structures-in-ruby/images/immutable-syntax-tree-488b8b56.png)
        - [tree-3](https://ev1stensberg.files.wordpress.com/2016/07/structural-sharing.png?w=776)
        - [tree-5](http://2.bp.blogspot.com/_r-NJO1NMiu4/TRA69XdCU8I/AAAAAAAAAnM/Re0VElAeLc4/s1600/ds_2_new.gif)

- What about hash maps?
    - mutable hash maps use an integer hash code and a big array indexed by the hash code
        - copying the array on every modification would be too slow
        - but hash code can be stored more efficiently in a trie
    - This image of a string trie illustrates how a trie can be built up from pieces of a string
        - [string trie](https://upload.wikimedia.org/wikipedia/commons/thumb/b/be/Trie_example.svg/1200px-Trie_example.svg.png)
    - Break integer hash into parts of 4 bits each
        - 0x1234 breaks into parts 1, 2, 3, and 4
        - build tree of arrays
            - each level in tree use appropriate part (first at root, second at depth 1, etc)
    - Images illustrating hash array mapped trie concepts
        - [2 bit hamt path](http://moaazsidat.com/assets/react-immutable/02-BitmappedArrayTrie.gif)
        - [sort-of-hamt](https://examples.javacodegeeks.com/wp-content/uploads/2016/07/trie1.jpg)
        - [hamt-1](https://idea.popcount.org/2012-07-25-introduction-to-hamt/439c703a1efeb7c39bf2d48839659eca.svg)
    - Javimmutable uses 32 element arrays (5 bits per part) at each level of tree
        - Don't you wind up with a ton of mostly empty arrays?
        - no! - create bit mask with each bit representing an array index
            - 32 bit int mask maps to 32 indexes in array, tree always max depth 7 (root has only 2 bit mask)
            - create array with size == number of 1 bits in mask
            - first 1 bit in mask is index of first element in array, etc
            - modern CPUs have native instruction to determine first 1 bit in an int
            - java provides native method to expose that instruction
        - [performance numbers](https://github.com/brianburton/java-immutable-collections/wiki/Comparative-Performance)

