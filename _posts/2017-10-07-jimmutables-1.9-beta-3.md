---
title:  "Javaimmutable Collections 1.9-beta-3"
date:   2017-10-07 21:46:00 -0400
categories: javimmutable release update
---
## Releasing 1.9

The last planned Java 6 compatible version of Javimmutable Collections should be officially released in a week or so.  This release will have the significantly faster B-Tree implementation of JImmutableRandomAccessList as well as all of the new collection types developed by Angela Burton.  Pre-release testing will use the vastly improved stress test tool also developed by Angela.

Stress tests have been running clean for days.  Unit tests and coverage all look good.  I'd like to improve the project's web site  prior to making a production release but we'll see what time allows.

This release has been long in coming but will be a great improvement in usability.  The new collection classes Angela added will be a big help.  The JImmutableMultiset is particularly nice when you need to count distinct objects.  It interacts nicely with normal sets as well as providing its own enhanced functionality.  We've needed a JImmutableSetMap for a long time now too.

Actually releasing the beta proved to be a bit of a challenge.  The **long** time elapsed between 1.8 and today led to a lot of rust in my release process.  Also at some point I'd made the questionable decision to delete my entire `~/.m2` directory to clean up my maven cache.  Nothing like a clean sweep to clear things up, right?  Well maybe but it meant I'd also deleted my settings.xml and settings-security.xml files.  To top if off the newer versions of the sonatype release plugins, the now incredibly picky javadoc plugin, and various other fun changes led to there being a beta-3 release rather than just a beta-1.  No code changes in those betas, just retries after failed releases.


## Starting 2.x Development

The next step for the library is to update it for Java 8.  This will include some useful features:

- All collections will have Java 8 `stream()` methods added.  These will allow Java 8 developers to use standard stream APIs with immutable collections.
- Collector classes will be added for all of the collection types.  This will make it easy to use streams to construct immutable collections.
- Not Java 8 specific but I'm considering adding java serialization support to the collections.  This is low priority but could be added if users express any interest.
- I'd like to switch to shorter interface names for the collections.  JImmutableRandomAccessList is a long name.  Maybe something like JiRList instead?  I'd have to do this in a completely backward compatible way of course.

## Another Project Idea

I've also been considering adding a separate add-on library for [Jackson](https://github.com/FasterXML/jackson) to allow JImmutables to be used directly as JSON properties in objects.  This could be very handy when creating immutable JSON capable beans.
