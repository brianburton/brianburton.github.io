---
title:  "Spring Coding"
date:   2020-04-08 12:00:00 -0400
categories: javimmutable
---
## Spring Coding

It seems my open source work is seasonal.  At least the frequency of the productivity bursts comes about as often as the seasons do (a few times a year).  Given all that's going on in the world the last couple of months and it's impact on my consulting work this seemed a great time to launch into action again.

The focus this time is to continue to improve memory footprint and overall performance of the key components of the Java Immutable Collections project.  This work really began in response to the very informative benchmarks posted by Albul in an issue on the java-immutable-collections project.  They showed that the library compared very favorably indeed to the alternatives.  They also provided a way to measure the effect of any refactoring and design overhaul I might try.

### Hash Collision Handling

The first target for improvement was JImmutableHashMap.  Albul's program indicated that this class used substantially more memory than the standard Java HashMap (7,375,600 bytes vs 5,846,608).  Reviewing the code I found some areas for improvement.

All hash map nodes stored their keys and values inside of "collision set" instances.  These were either little lists or tree maps depending on whether or not the keys were Comparable.  While there is nothing wrong with that it did leave room for improvement.

With a decent hashCode() method the number of collisions should be minute in most cases.  Especially since JImmutableHashMap doesn't need to `mod` the hash code down to fit inside an array.  Unlike with a standard hash table the entire hash code is always available to distinguish between keys.  It's vital to have a strategy to deal with collisions but that strategy should be needed infrequently in practice.

To exploit this fact I simply added new leaf node implementations specialized for a single key/value pair.  This is always used when adding a new key with no collision.  The leaf contains just the key and value as fields within itself so no extra collision set instances are needed.  Obviously this reduced both the CPU overhead (fewer objects to interact with) and memory footprint.

### Hash Sets

The next target for improvement was JImmutableHashSet,  Sets have always used maps internally in their implementation.  This made the code more maintainable but at the price of memory footprint and performance.  Since I use hash sets pretty often I decided to go ahead and write a custom hash set implementation.  This both improved performance and reduced memory footprint.  Win - win.

## Update

These changes were incorporated into the 3.1.0 release on April 16.

