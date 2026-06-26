---
title: "Comprehensive JVM Primitive Hashtable Benchmarks 2026"
date: 2026-06-18
draft: false
ShowToc: true
TocOpen: true
math: true
charts: true
---

## Introduction

Hashtables are one of the building blocks of computer science, and deservedly get a lot of attention - but less so
within the JVM ecosystem. Part of this may simply be that most Java code is not written with performance top of mind,
or that the default JRE implementation is all-around quite good - reasonably efficient, resistant against attacks, 
etc... I'm interested however in focusing entirely on performance today.

For today, we're interested specifically in hashtables for primitive values. By focusing on values rather than
references we'll entirely ignore the cost of heap allocation, pointer dereferencing, GC pressure, and also come up 
with an analysis that will hopefully continue to hold weight with upcoming
[Valhalla](https://openjdk.org/projects/valhalla/) changes to the JVM.

Hashtables that store values rather than references can provide a range of large performance benefits within the JVM,
such as no boxing, no pointer indirection, better cache locality, etc... The JVM ecosystem has accumulated a handful of
libraries that address this by providing hashtables backed by primitive arrays. The problem is that the performance 
tradeoffs and design choices between them are neither obvious nor well-documented anywhere. Many different choices 
are made around hash functions, collision resolution strategies, load factors, and those choices produce
meaningfully different performance profiles.

We'll benchmark and analyze the following libraries (links go to more detailed information lower down this page):

* [JRE](#jre)
* [FastCollect](#fastcollect) (disclaimer: I am the author of FastCollect)
* [Fastutil](#fastutil)
* [Eclipse](#eclipse)
* [AndroidX](#androidx)
* [Trove](#trove)
* [Koloboke](#koloboke)
* [HPPC](#hppc)
* [Agrona](#agrona)
* [PrimitiveCollections](#primitive-collections)

You may note that several of the libraries benchmarked here are quite old and/or no longer maintained. The larger 
purpose of this post is not to determine which is fastest (well - we all know that's a kinda cute lie), but to 
investigate in detail the design choices that affect hashtable performance. Also note that the JRE is the only library
here truly suited for external use (when an attacker can control the table) - most of these libraries have abandoned any
pretense at resisting DoS attacks in favor of raw performance (both is rarely an option).

Full benchmarking code is available at https://github.com/sooniln/jvm-collections-benchmarks.

## Hashtable Design Choices

Before we begin, it's worth starting with a quick refresher on how hash tables work, and the various design choices that
have sprung up over the years. If you're already reasonably familiar with how hashtables work, feel free to skip 
straight to the [Libraries Under Test](#libraries-under-test) section, or the [Benchmark Setup](#benchmark-setup).

Pretty much all general purpose hashtables operate by mapping a very large universe of potential keys onto a much
smaller number of available slots (represented by in-memory data structure, almost always an array). As a trivial
example, if we're using 32-bit integers as keys, we would need to map all the ~4 billion possible integers onto the much
smaller array in memory (for example a 128 slot array perhaps, if we're only expecting to map 100 integers total). This
is accomplished via [hash functions](https://en.wikipedia.org/wiki/Hash_function).

Now by definition, once we've computed such a mapping there may be collisions - two different keys that are mapped 
by a hash function to the same slot. The second responsibility of the hash table is thus to handle collisions, and
store multiple keys that map to the same slots. This brings us to our first design decision,
[open addressing](https://en.wikipedia.org/wiki/Open_addressing) vs
[separate chaining](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining).

### Open Addressing vs. Separate Chaining

The first architectural fork in hashtable design is how collisions are handled when two keys map to the same slot.

**Separate chaining** associates a separate collection of entries with each slot. All collections thus are stored 
within the same collection. The advantage here is simplicity - it's trivial to add and remove things, and the selection
of different collection types modifies performance. The most common collection used here is a linked list, but other
options are possible (the JRE HashMap starts with a linked list for example, but converts to a red-black tree if the
list grows too large). The downside is an additional indirection to enter the collection, and the cost of lookup within
the collection (pointer traversal such as in a linked list is very cache unfriendly for example).

**Open addressing** stores all entries directly in the contiguous backing array. When a slot is occupied, the algorithm
probes for the next candidate slot according to some probing strategy (discussed in more detail below). Lookups and 
insertions are often likely to stay near the original slot (cache friendly), which is why open addressing dominates 
value hashtables (which don't have the built-in pointer dereferencing cost of references). Deletion in an open 
addressed table requires extra thought, and is discussed in more detail below.

Every library in this benchmark except the JRE HashMap uses open addressing.

### Load Factor

The [load factor](https://en.wikipedia.org/wiki/Hash_table#Load_factor) of a hashtable is the ratio of occupied 
slots to total capacity. At load factor 0.5, half the backing array is empty. At 0.9, only one slot in ten is free.
Most hashtables have a maximum load factor - when they reach that limit the backing array is expanded to reduce the
load factor again. This provides a convenient upper bound on performance, and allows the hashtable to scale memory 
usage with the number of entries.

Higher load factors conserve memory but increase expected probe sequence length, which grows non-linearly as the table
fills. For a uniformly random hash function and a simple linear probe, expected probe length at load factor α is
approximately:

* Successful lookup (hit) ≈ $\frac{1}{2}\left(1 + \frac{1}{1-\alpha}\right)$
* Unsuccessful lookup (miss) ≈ $\frac{1}{2}\left(1 + \frac{1}{(1-\alpha)^2}\right)$

At α=0.5 this means ~1.5 probes for hits and ~2.5 for misses. At α=0.75 that's ~2.5 probes for hits and ~8.5 for misses.
And at α=0.9: ~5.5 for hits, ~50.5 for misses! This is a very simple calculation, assuming a perfect hash function and
the simplest possible version of linear probing, but it is useful for understanding general probe length behavior.

### Backing Array Sizing

Open-addressed hashtables also need to decide how large of an array to use to hold some number of entries at a given 
load factor. There are two main strategies.

**Power-of-two sizing** uses table sizes that are always powers of two: 16, 32, 64, 128, and so on. This is primarily
for performance reasons:

1) Multiplying/dividing by two is a simple shift operation.
2) Converting a hash to a slot index can be done with a single AND rather than a modulo operation:

```java

int capacity = 256;      // 0b100000000
int mask = capacity - 1; // 0b011111111
int slot = hash & mask;  // slot is now in [0, 256)
```

A binary AND should take 1 cycle on any CPU these days - compared to a modulo operation which requires integer division
and can take 10-40+ cycles.

**Prime sizing** keeps the table size as a prime number: 17, 37, 79, etc... Converting a hash to a slot now requires a
modulo, and is thus slower. The case for prime sizing is that because the hash cannot have any hidden common factors
with the prime table size (by definition), this results in a much more even spread of keys within the table and thus 
reduces clustering. Less clustering reduces average probe length, which reduces lookup times. There is one additional
advantage to prime sizing - for non-linear probing schemes such as quadratic probing (discussed later) it is much 
easier to ensure no infinite loops in probing (every slot will be visited by the probe before repeating).

The simple implementations of both of these schemes requires a large re-allocation and complete rehash (re-insertion 
of all elements) whenever the hashtable runs out of room. There are scenarios where this large "pause the world" 
type of logic may be unacceptable, and so there are also
[variations](https://en.wikipedia.org/wiki/Hash_table#Alternatives_to_all-at-once_rehashing) that allow for more 
gradual resizing over time - usually at the expense of performance.

### Hashtable Memory Layout

Hashtables need to store both keys and value. There are generally two simple for keys and values:

**Parallel arrays** maintain one array for keys and a separate array for values:

* keys:   \[k0]\[k1]\[k2]\[k3]...
* values: \[v0]\[v1]\[v2]\[v3]...

**Interleaved storage** packs key-value pairs together:

* table: \[k0]\[v0]\[k1]\[v1]\[k2]\[v2]\[k3]\[v3]...

Note that interleaved storage can waste more space in order to achieve memory alignment if the keys and values are of
different sizes.

The advantage of separate arrays is that when a hashtable is probing through keys, more keys will fit in a single cache 
line, resulting in less cache misses. However once you've located the correct key, you still need to load the value -
with separate arrays the value is located somewhere else entirely, and you're incurring another possible cache miss to
load the value. Using interleaved storage means that once you locate the key, the value is immediately adjacent in
memory - no extra load necessary. But while probing, every cache line can now hold less keys, and thus more cache
misses are incurred...

#### Metadata

Rather than storing keys/values directly in the primary hashtable arrays, there are other possibilities as well:

For example,
[Robin Hood](https://en.wikipedia.org/wiki/Robin_Hood_hashing) hashtables may store probe sequence lengths (PSLs - how 
far a key is located from the slot it hashed too), and [Swiss tables](https://abseil.io/about/design/swisstables) store
key fingerprints. Given that that metadata is usually substantially smaller than the key size, metadata cannot easily
be interleaved with keys. If probing now requires accessing both metadata and keys this will incur many more cache
misses, so when metadata is used probing will almost always attempt to only access metadata, and access keys only when
absolutely necessary.

There are also hashtables that store indirect indices which point into separate key/value storage. This allows 
key/value storage to be much denser than the main hashtable, potentially saving space, at the cost of an additional 
level of indirection. 

Finally, there is one more advantage to metadata - it can help with representing empty slots, which will be discussed 
in the next section.

### Empty Slots

We've discussed key arrays, value arrays, interleaved key/value arrays, and separate metadata arrays. No matter which
is used however, a hashtable needs some way to indicate an empty slot, so that it know where it can insert new 
entries or when it can stop probing during a lookup. The problem of course, is that there is no guarantee any  
particular key value you might choose to represent "unoccupied" won't be inserted! There are naturally many possible 
solutions to this.

1. Enforce that the user provides an illegal key value which will always represent empty - the user can never insert
   that key into the map. Unworkable for any general purpose hashtable.
2. Use a separate variable to track whether the empty key value has been added to the map and what its value is. In
   every hashtable operation, check if the given key is the same as the empty key, and if so special-case the operation
   to use the separate variable instead of looking in the table.
3. Always allocate an array with 1 additional slot at the end, which is used to hold the value of the empty key. 
   In the rest of the array, the empty key means an empty slot.
4. If the user attempts to insert the empty key, choose a new empty key which is not present in the table, and update
   the table to use the new empty key before continuing the insertion.
5. If using a metadata array, represent the empty slot in the metadata array. 

There are of course potential performance implications to all of these choices. If you need to add special cases you'll
add to code size - too large of a code block can prevent some JVM C2 compiler optimizations and also incur more code
cache misses. If you need to check a member variable this is an additional load and register usage. Too much register
usage can cause register spilling and commensurate performance impacts on the hot path. No matter the choice there's
always a variety of consequences to think through, and pretty much no way to anticipate the impact except through a
concept we haven't mentioned at all yet (benchmarking).

### Probing Strategies

So far we've generally covered how data is represented internally, and how the map converts a hash into a table slot,
but what happens when that slot is already occupied (a hash collision)? In the case of separate chaining, we simply add
the new entry to the collection of entries already associated with the slot, but what about open addressing?

When a slot is occupied, the table must find another. The sequence of slots tried in the search for an open slot is
called the *probe sequence*, and its choice significantly impacts both throughput and cache behavior.

**[Linear probing](https://en.wikipedia.org/wiki/Linear_probing)** just tries consecutive slots: slot, slot+1, slot+2, 
and so on (wrapping around the table) until an open slot is found. It is maximally cache-friendly (sequential memory 
access) — but suffers from *primary clustering*: once a run of occupied slots forms, any key that hashes into that run 
lengthens it, creating a feedback loop that lengthens the run further and degrades performance as entries are put into 
slots further and further away from their home slot, and thus requiring longer and longer probe sequences on lookup.

```
State: [ ][ ][A][B][C][ ][ ]

Insert D (hashes to slot 2): probe 2→3→4→5, insert at 5.
State: [ ][ ][A][B][C][D][ ]

Insert E (hashes to slot 5): probe 5→6, insert at 6. Cluster continues to grow...
State: [ ][ ][A][B][C][D][E]
```

**[Quadratic probing](https://en.wikipedia.org/wiki/Quadratic_probing)** uses offsets slot+0², slot+1², slot+2²,  
slot+3²... (the probe number squared). It avoids primary clustering but can instead produce secondary clustering 
(keys with the same initial slot follow identical probe sequences) and over longer probe sequences results in more 
cache misses than linear probing.

**[Double hashing](https://en.wikipedia.org/wiki/Double_hashing)** uses a second hash function to calculate the stride
size based on the key: slot, slot+stride, slot+2\*stride, slot+3\*stride, etc. It avoids both primary and secondary 
clustering - the cost is computing two hash functions and the non-sequential memory access pattern which also 
results in additional cache misses.

**[Robin Hood hashing](https://en.wikipedia.org/wiki/Hash_table#Robin_Hood_hashing)** is a refinement of linear probing
that swaps entry positions (swapping a "poorer" \[further from it's home slot\] entry with a "richer" \[closer to 
it's home slot\] entry - hence Robin Hood) during insertion to reduce probe length variance. Entries that are "rich" 
and landed near their ideal slot yield their position to "poor" entries that have traveled further. This equalizes 
probe lengths and allows much higher load factors without the worst-case blow-up of plain linear probing (Robin Hood 
hashtables are often run at load factors of .9 without much performance impact).

And finally of course, you can have arbitrarily complex combinations of any probing scheme. A table could linear probe
for the first N elements, then switch to quadratic probing, or switch to linear probing with a different hash function, 
etc... The advantage to more complex probing schemes is that you can attempt to equalize the various pros and cons 
of each. For example, by starting with linear probing you can take advantage of its cache friendly behavior 
initially - linear probing for a cache line distance to avoid extra cache misses - then switch to quadratic probing 
when you're going to incur a cache miss anyway. The downside of these arbitrarily complex combinations is of course 
code complexity - but also entry removal, which can become very difficult.

### Entry Removal

Now that we've walked through the process of adding entries to the hashtable, how do we remove entries? Deleting a key 
from an open-addressed map is more complex than insertion. We can't simply find the slot in the array and clear 
it - this would break the invariants various probing algorithms rely on to function. Clearing the slot would 
leave a gap in a probe sequence, causing lookups for later keys to terminate early at the gap and incorrectly report 
a key as not present in the table.

One standard solution is **[lazy deletion](https://en.wikipedia.org/wiki/Lazy_deletion)**: rather than clearing the 
slot, mark it with a special DELETED value (called a tombstone). Future lookups probe past tombstones, and future 
insertions can reuse them. Tombstones are very easy to implement, but can accumulate over time. A map with many 
insertions and deletions will fill up with tombstones, which probe sequences must scan through even though they
represent no data. This degrades all operations over time. Most implementations address this by triggering a rehash 
when tombstones exceed some fraction of capacity, but this means delete-heavy workloads periodically pay a 
full-table rehash cost.

**Backward-shift deletion** is an alternative that avoids tombstones entirely. After clearing a slot, the algorithm 
looks at the next slot and checks if its occupant could shift back one position (i.e., its probe distance is greater
than zero — it's not at its ideal slot). If so, it shifts back. This continues until an empty slot or a slot whose 
occupant is already at its ideal position is reached.

```
Before (d = displacement from ideal):
[      ][A(d=0)][B(d=1)][C(d=2)][      ]   

Delete A:
[      ][      ][B(d=1)][C(d=2)][      ]

Shift B back (d=1, shift allowed):
[      ][B(d=0)][      ][C(d=2)][      ]

Shift C back (d=2, shift allowed):
[      ][B(d=0)][C(d=1)][      ][      ]
```

The result is a table with no tombstones and no gaps — every probe sequence remains contiguous. Lookup and insertion
performance doesn't degrade after heavy deletion. The cost is that deletion is slightly more expensive, and that
(1) element displacement must be linear (2) an element's displacement must be calculable. Robin Hood hashing tends to 
maintain this naturally, so backwards shift deletion is commonly used with Robin Hood hashing, but it is 
possible with other variations as well.

### Hash Functions And Mixing: Avalanche and Bias

So far we've discussed the simple mechanics of how a hashtable works, but avoided thinking too much about the hash 
function (often assuming a "perfect" hash function). Real life may have problems with these assumptions. Let's start 
with, what does it even mean to be a perfect hash function?

For a 
primitive integer map, the key itself is an 
integer. A naive implementation
uses that integer directly as the hash. This is what Trove does for `int` keys:

```java
// Trove TIntIntHashMap (via identity hash)
int index = key & mask;  // only low bits determine slot — terrible for sequential keys
```

The problem is that low-order bits of sequential integers (0, 1, 2, 3...) are
structured: with a power-of-two table of size 16, keys 0–15 each hit a different
slot perfectly, but keys 16–31 collide with all of them. This clustering behavior
is why Trove uses prime tables and double hashing to compensate.

A good hash function *mixes* or *[avalanches](https://en.wikipedia.org/wiki/Avalanche_effect)* the input bits: every bit of the output
depends on every bit of the input (or enough of them that bias in the lower bits is
eliminated). For power-of-two tables where indexing is `hash & (n-1)`, this means
the lower bits of the hash must be well-mixed.

The most common approach in this benchmark family is **[Fibonacci/phi multiplicative
hashing](https://en.wikipedia.org/wiki/Hash_function#Fibonacci_hashing)** followed by a xor shift:

```java
// fastutil HashCommon.mix(), Koloboke, HPPC BitMixer.mixPhi(), Speiger HashUtil.mix()
// (all four libraries use this identical function — Speiger's Javadoc credits Koloboke)
static int mix(int x) {
    final int h = x * 0x9E3779B9;  // multiply by golden ratio (2^32 / φ)
    return h ^ (h >>> 16);          // fold high bits into low bits
}
```

The constant `0x9E3779B9` is the 32-bit approximation of 2³²/φ (φ = golden ratio ≈
1.618). Multiplying by it scatters inputs across the full 32-bit range in a Fibonacci
sequence pattern, which distributes keys uniformly across power-of-two buckets. The
xorshift-right-16 folds the high bits (which received most of the mixing energy from
the multiplication) back into the low bits, which are the bits used for bucket
selection.

AndroidX uses a different constant, taken from MurmurHash:

```kotlin
// AndroidX ScatterMap.kt / IntIntMap.kt
internal inline fun hash(k: Int): Int {
    val hash = k.hashCode() * 0xcc9e2d51.toInt()  // MurmurHash C1
    return hash xor (hash shl 16)                  // note: left shift (not right)
}
```

The left-shift variant (`shl` instead of `ushr`) folds low bits into high bits — the
opposite direction from the phi-mix. This makes sense for the Swiss Table design,
where H1 is the *high* bits (used for the initial probe position) and H2 is the *low*
bits (stored as the fingerprint).

Agrona applies a stronger two-round mix for `int` keys and a three-round mix for
`long` keys, at the cost of more operations per hash call:

```java
// Agrona Hashing.java
public static int hash(final int value) {
    int x = value;
    x = ((x >>> 16) ^ x) * 0x119de1f3;  // round 1
    x = ((x >>> 16) ^ x) * 0x119de1f3;  // round 2
    return (x >>> 16) ^ x;
}

public static int hash(final long value) {
    long x = value;
    x = (x ^ (x >>> 30)) * 0xbf58476d1ce4e5b9L;  // round 1 (Murmur3 finalize)
    x = (x ^ (x >>> 27)) * 0x94d049bb133111ebL;  // round 2
    x = x ^ (x >>> 31);                          // round 3
    return (int)x ^ (int)(x >>> 32);
}
```

The two-round int mix is a variant of Murmur3's finalizer with better avalanche
properties than the single-multiply phi-mix, but more CPU cycles per call. Whether
this pays off in benchmark throughput depends on the key distribution: for
adversarial or clustered inputs it helps; for random inputs, the cheaper phi-mix is
often sufficient.

Eclipse Collections uses two independent 32-bit spread functions, applied separately
to select the primary and secondary probe positions:

```java
// Eclipse Collections SpreadFunctions.java
private static int thirtyTwoBitSpread1(int code) {
    code ^= code >>> 15;
    code *= 0xACAB2A4D;
    code ^= code >>> 15;
    code *= 0x5CC7DF53;
    code ^= code >>> 12;
    return code;
}

private static int thirtyTwoBitSpread2(int code) {
    code ^= code >>> 14;
    code *= 0xBA1CCD33;
    code ^= code >>> 13;
    code *= 0x9B6296CB;
    code ^= code >>> 12;
    return code;
}
```

These are higher-quality mixes than the single-multiply phi approach (two
multiply-xorshift rounds each), at higher cost. The use of two independent functions
is consistent with a double-hashing-like probing strategy rather than pure linear
probing.

**Implication for benchmarks**: hash function quality becomes visible in performance
degradation under adversarial or clustered key distributions. For uniformly random
keys, all mixing functions produce similar collision rates. The benchmark uses both
sequential and random key distributions to surface this difference.

#### Knuth Multiplicative Hashing

## Libraries Under Test



| Library              | Storage Schema                                                                  |
|----------------------|---------------------------------------------------------------------------------|
| JRE                  | Node pointers with boxed keys/values (can form a linked list or red-black tree) |
| FastCollect          | Metadata + Interleaved if key/value size matches, Parallel otherwise            |
| Fastutil             | Parallel                                                                        |
| AndroidX             | Metadata + Parallel                                                             |
| Trove                | Metadata + Parallel                                                             |
| Koloboke             | Interleaved if key/value size matches, Parallel otherwise                       |
| Eclipse              | Parallel                                                                        |
| HPPC                 | Parallel                                                                        |
| Agrona               | Parallel                                                                        |
| PrimitiveCollections | Parallel                                                                        |

| Library              | Probing Strategy                                                                   |
|----------------------|------------------------------------------------------------------------------------|
| JRE                  |                                   |
| FastCollect          | Robin Hood                                                                         |
| Fastutil             | Linear probing                                                                     |
| AndroidX             | Quadratic probing (each probe checks a group of 8 linear entries)                  |
| Trove                | Double hashing                                                                     |
| Koloboke             | Linear probing                                                                     |
| Eclipse              | Combination (linear probing on hash1 -> linear probing on hash2 -> double hashing) |
| HPPC                 | Linear probing                                                                     |
| Agrona               | Linear probing                                                                     |
| PrimitiveCollections | Linear probing                                                                     |

### JRE

Java's standard library hashtable uses separate chaining: each bucket holds a linked list that is automatically 
promoted to a red-black tree once it grows too large, bounding worst-case lookup from O(n) to O(log n) per bucket 
under adversarial key distributions. All keys are stored as boxed `Integer` or `Long` objects on the heap; every 
primitive key thus incurs an allocation and a pointer indirection on access.

* **Storage Schema**: Power-of-two sized, array of node pointers holding boxed keys/values (can form a linked list or 
  red-black tree)
* **Probing Strategy**: Separate chaining (linked list/red-black tree)
* **Deletion Strategy**: Removal from linked list/red-black tree
* **Default Load Factor**: 75%

**Integer key hash finalizer**: The hash finalizer performs minimal mixing, relying on the red-black tree fallback 
behavior to prevent pathological lookups.

```
key ^ (key >>> 16);
```

### FastCollect

*Disclosure: I am the author of FastCollect.*

FastCollect is a Kotlin Multiplatform primitive collections library supporting JVM, JS, and native targets.

**Source:** [sooniln/fastcollect](https://github.com/sooniln/fastcollect)  
**Maven:** `io.github.sooniln:fastcollect-kotlin-jvm:2.0.0`
**Last release:** June 2026

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear Probing (Robin Hood)
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 90% (100% for very small tables)

**Integer key hash finalizer**: FastCollect makes a bet that keys will tend to either fall within the hashtable size 
bound, or outside of it, and that the branch in the finalizer can thus be easily predicted. If this assumption is 
not true, performance will suffer (revisited later in the benchmarking section). When the key is not within the size 
bound uses Knuth multiplicative hashing with constant shift and pre-mixing step.

```
if (key < size) {
    return key
} else {
    var h = key ^ (key >>> 16)
    h *= 0x9E3779B9 // PHI
    return h ^ (h >>> 16)
}
```

### Fastutil

One of the most comprehensive and widely-adopted JVM primitive collections libraries, Fastutil is very well known.

**Source:** [vigna/fastutil](https://github.com/vigna/fastutil)  
**Maven:** `it.unimi.dsi:fastutil:8.5.18`  
**Last release:** October 2025

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear Probing
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

### AndroidX

Google's Jetpack collection utilities library, now written in Kotlin and available as a Kotlin Multiplatform artifact.
The implementation is a Swiss Table design that uses SWAR (SIMD within a register) techniques to evaluate groups
of 8 entries at a time. AndroidX libraries are primarily intended for arm64, though they are distributed as 
multi-platform. Since benchmarks were run on amd64, care should be taken not to generalize the results.

**Source:** [android.googlesource.com](https://android.googlesource.com/platform/frameworks/support) ([GitHub mirror](https://github.com/androidx/androidx))  
**Website:** [developer.android.com/jetpack/androidx/releases/collection](https://developer.android.com/jetpack/androidx/releases/collection)  
**Maven:** `androidx.collection:collection-jvm:1.6.0`  
**Last release:** March 2026

* **Storage Schema**: Power-of-two sized, metadata array of 1 byte/entry, parallel arrays for keys/values
  * **Metadata Schema**: Stores low 7 bits of original hash as the metadata fingerprint, the high 25 bits are used to
    drive the probe offset
* **Probing Strategy**: Quadratic Probing (where each probe checks a group of 8 linear entries)
* **Removal Strategy**: Tombstones + rehash to remove tombstones at ~78% capacity
* **Default Load Factor**: 87.5%

**Integer key hash finalizer**: Murmur3 finalizer. Note that the left shift instead of right shift would seem to
concentrate hash entropy further in the upper bits. Keys with no entropy in the low bits we would expect to hurt
performance (and indeed, benchmarking results confirm this).

```
var h = key * 0xCC9E2D51 // MurmurHashC1
return h ^ (h << 16)
```

### Trove

GNU Trove uses open addressing with double hashing: rather than a fixed linear step, each key computes a secondary
probe stride from its primary hash, so colliding keys scatter along independent paths and form shorter clusters than
linear probing would.

**Source:** [bitbucket.org/trove4j/trove](https://bitbucket.org/trove4j/trove)
**Maven:** `net.sf.trove4j:core:3.1.0`
**Last release:** August 2018 (due to module split - no active development since ~2012 apparently)

* **Storage Schema**: Prime sized, metadata array to track slot empty state + parallel arrays for keys/values
    * **Metadata Schema**: Only tracks empty state, which means every probe step has to check both metadata array and
      key arrays - would expect this to hurt performance.
* **Probing Strategy**: Double hashing
* **Removal Strategy**: Tombstones, rehash if there are no free slots (only tombstones) left, also rehash if removal 
  allows backing array to shrink to the previous prime size
* **Default Load Factor**: 75%

**Integer key hash finalizer**: The identity hash — no bit mixing is applied; distribution quality relies entirely on
the double-hashing probe and prime sized arrays to counteract clustering.

```
// double hashing
// slot - identity hash
return key

// stride
return 1 + (key % (length - 2))
```

### Koloboke

A high performance collections library from Roman Leventov, appears to have been designed with HFT in mind. Makes use
Unsafe APIs for performance.

**Source:** [leventov/Koloboke](https://github.com/leventov/Koloboke)
**Maven:** `com.koloboke:koloboke-impl-jdk8:1.0.0`
**Last release:** May 2016

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```
var h = x * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

### Eclipse

Originally the Goldman Sachs Collections library, donated to the Eclipse Foundation in 2012. Eclipse forces a load 
factor of 50% - much lower than any other library in this benchmark and which cannot be adjusted. Beware assuming 
an apples to apples comparison. Some experimentation with running other libraries with a 50% load factor indicates 
they are competitive and may out-perform Eclipse, but full benchmarking has not been performed.

**Source:** [eclipse/eclipse-collections](https://github.com/eclipse/eclipse-collections)  
**Website:** [eclipse.dev/collections](https://eclipse.dev/collections/)  
**Maven:** `org.eclipse.collections:eclipse-collections:13.0.0`
**Last release:** December 24, 2024

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Combination (linear probing on hash1 for half a cache line -> linear probing on hash2 for 
  half a cache line -> double hashing)
* **Removal Strategy**: Tombstones - tombstones count towards rehashing load factor
* **Default Load Factor**: 50%

**Integer key hash finalizer**: 3 separate hash finalizers for the various probing steps. It's not entirely clear where 
the constants come from.

```
// hash1 - identity hash
return key

// hash2 - murmur3 style finalizer
var h = key ^ (key >>> 14)
h *= 0xBA1CCD33
h ^= h >>> 13
h *= 0x9B6296CB
return h ^ (h >>> 12)

// double hashing
// slot
var h = key ^ (key >>> 15)
h *= 0xACAB2A4D
h ^= h >>> 15
h *= 0x5CC7DF53
return h ^ (h >>> 12)

// stride - odd stride ensures every slot is hit with power-of-two sized array
return Integer.reverse(hash2(key)) | 1
```

### HPPC

High Performance Primitive Collections, maintained by Carrot Search. Iteration uses a randomized starting offset (seeded per-instance) to prevent
consistent worst-case traversal patterns. The `mixPhi(long)` variant XOR-folds its result to `int` directly, unlike fastutil's `mix(long)` which preserves the full 64-bit result.

**Source:** [carrotsearch/hppc](https://github.com/carrotsearch/hppc)  
**Website:** [labs.carrotsearch.com/hppc.html](https://labs.carrotsearch.com/hppc.html)  
**Maven:** `com.carrotsearch:hppc:0.10.0`  
**Last release:** June 2024

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

### Agrona

Agrona is Real Logic's utility library, developed primarily to support the [Aeron](https://github.com/aeron-io/aeron)
messaging system. Agrona enforces every map must have an unrepresentable value passed in on initialization (to mark 
empty slots), thus Agrona is incapable of representing the full range of values.

Note: Since Agrona does not have a Long → Int map, benchmarking uses Long → Long maps instead, and widens values. Be 
wary of apples to apples comparisons.

**Source:** [aeron-io/agrona](https://github.com/aeron-io/agrona)  
**Maven:** `org.agrona:agrona:2.4.1`  
**Last release:** April 2026

* **Storage Schema**: Power-of-two sized, interleaved arrays for keys/values (Agrona does not support key and values 
  with different sizes)
* **Probing Strategy**: Linear Probing
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 65%

**Integer key hash finalizer**: Two-round XOR-shift/multiply from Chris Wellons' Hash Prospector.

```
var h = key ^ (key >>> 16)
h = h * 0x119DE1f3
h = h ^ (h >>> 16)
h = h * 0x119DE1f3
return h ^ (h >>> 16)
```

### Primitive Collections

PrimitiveCollections is a library explicitly aimed at duplicating and outperforming fastutil.

**Source:** [Speiger/Primitive-Collections](https://github.com/Speiger/Primitive-Collections)  
**Maven:** `io.github.speiger:Primitive-Collections:1.0.0`  
**Last release:** May 2026

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear Probing
* **Removal Strategy**: Backwards shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

## Benchmark Setup

* No CPU pinning, ran on idle machine.
* 6 Core AMD Ryzen 5 9600X - Windows 11

Benchmarks were run until standard deviation to mean ratios were at a reasonable level (where possible - there were a
few benchmarks that still did not converge nicely after a reasonable number of re-runs), then median was used to
score in order to avoid outliers.

Two types of maps tested:

Integer → Integer
* Same size for keys/values allows for interleaved memory layouts without penalty.
* Smaller keys pack more into cache.

Long → Integer
* Different size for keys/values prevents interleaved layouts.
* Larger keys increase cache pressure.

The hashtables under test come with a variety of different default load factors. We face a choice of whether to try and
force similar load factors for a more apples-to-apples comparison, or use the default load factors which may 
unfairly advantage some implementations. Load factors have become an important part of hashtable design - it's a 
valid choice to use a low load factor for performance and recover memory in other ways (by ensuring the load factor
only affects the smaller metadata array and packing keys/value tightly outside the main array for example). However, 
none of the hashtables benchmarked here appear to be applying any such memory optimizations, and in the interest of 
similar comparisons we try to enforce that every hashtable has a load factor of at least 75%.

| Library              | Default | Benchmarked | Adjustable |
|----------------------|---------|-------------|------------|
| JRE                  | 75%     | 75%         | Yes        |
| FastCollect          | 90%     | 90%         | No         |
| Fastutil             | 75%     | 75%         | Yes        |
| AndroidX             | 87.5%   | 87.5%       | No         |
| Trove                | 50%     | 75%         | Yes        |
| Koloboke             | 66.7%   | 75%         | Yes        |
| **Eclipse**          | 50%     | **50%**     | No         |
| HPPC                 | 75%     | 75%         | Yes        |
| Agrona               | n/a     | 75%         | Yes        |
| PrimitiveCollections | 75%     | 75%         | Yes        |

### Key Selection and Ordering

Because hash functions have such a large effect on the performance of hash tables, it's important to test with data 
that can represent real world scenarios. We specify 3 key orders to test under:

* **random**: Keys are selected at random from the entire universe of possible keys. This represents scenarios where
  only a subset of keys are stored - for example if a hashtable is being used as a cache.
* **lowBits**: Keys are selected from 0 → table size and shuffled randomly. This has the effect that only the least 
  significant bits of the key tend to be set, and most significant bits are all zero. This represents scenarios where 
  almost the entire key set is stored.
* **highBits**: The same as lowBits, but every key is bit-reversed, so that the most significant bits tend to be set,
  and the least significant are all zero. This represents two scenarios - (1) an easy adversarial attack (since the 
  vast majority of hashtables are power-of-two sized, we know that they will usually truncate high bits in order to 
  calculate a slot - without good bit-mixing properties these tables may be vulnerable) (2) structures that have been 
  bit-packed into keys and thus have unusually structured bits (for example, a Point class can trivially be encoded as 
  a 64 bit value).

In practice, I suspect (with no evidence other than gut feeling) that random and lowBits are likely to represent the 
majority of real world scenarios in the JVM ecosystem.

### Benchmark Selection

We benchmark the following operations:

* **getHit**: The cost of looking up a value for a key present in the hashtable. Note that since different 
  hashtables have different conventions for representing keys that are not present (some of which may give that 
  hashtable an unfair advantage), we use getOrDefault() or equivalent APIs for a fair comparison.
* **getMiss**: The cost of looking up a value for a key not present in the hashtable. Note that since different
  hashtables have different conventions for representing keys that are not present (some of which may give that
  hashtable an unfair advantage), we use getOrDefault() or equivalent APIs for a fair comparison.
* **putHit**: The cost of changing the value for a key present in the hashtable. When offered, this will always use 
  an API which does not return a value in order to find the cheapest possible cost.
* **putMiss**: The cost of assigning a value to a key not present in the hashtable. When offered, this will always use
  an API which does not return a value in order to find the cheapest possible cost.
* **removeAndPutMiss**: The cost of removing a key and then assigning a value to a different key that is not present in
  the table. The setup of JMH tests makes it difficult to accurately quantify the cost of just a remove operation - 
  this is a compromise.
* **forEach**: The cost of iterating over every key/value present in the hashtable using the fastest iteration 
  technique exposed by the hashtable.
* **naiveCopy**: The cost of copying the hashtable starting from empty and inserting elements until complete - 
  with no information given on the total number of elements to be expected, so no pre-allocation can be done.
* **preAllocatedCopy**: The cost of copying the hashtable directly as cheaply as possible (may just be a memcpy in the
  cheapest case).

### Major Caveats **PLEASE READ**

standard micros benchmark caveats

no benchmarking of tombstones

## Benchmark Results

Raw benchmark results can be downloaded [here](/libraries.csv).

While viewing graphs, you may use the button to switch the time axis between log scale and linear scale. Clicking on 
a library in the legend will hide/show the data for that library.

### Overall Results

Rather than forcing you to scroll past the excessive graphs and numbers below which go into individual scenarios, 
might as well cut to the chase, right?

### FastCollect Branch Prediction

### 50% Load Factor

### Int → Int

We start with the results of benchmarking Int → Int hashtables.

#### Int → Int / getHit

{{< benchmark-chart benchmark="IntMap.getHit" order="random" title="Int → Int / getHit — random keys" >}}

As this is the first graph we're looking at, note the distinct regime changes as the hashtable size increases past 
L1 and L2 cache size. For many of the power-of-two sized graphs we see the distinct sawtooth performance that comes 
from different load factors as the table is queried when more empty vs more full. The JRE boxed HashMap is on the 
slower side but remains reasonably competitive. Eclipse performs well with its maximum 50% load factor, though at 
the cost of almost double the memory usage in some cases.

{{< benchmark-chart benchmark="IntMap.getHit" order="lowBits" title="Int → Int / getHit — lowBits keys" >}}

Trove performs dramatically better on lowBits keys than on random keys. Recall that Trove uses the identity hash 
function and prime-sized tables - for lowBits keys the identity hash is pretty much a perfect hash function, which 
explains its excellent performance.

{{< benchmark-chart benchmark="IntMap.getHit" order="highBits" title="Int → Int / getHit — highBits keys" >}}

AndroidX was performing on the slower side, but still reasonably for random and lowBits keys - however it completely 
falls apart on highBits keys, especially at smaller sizes. ~1500 ns to find any key in a 4k sized table is very odd 
and troubling. Recall that AndroidX uses the low 7 bits as the fingerprint - it's likely that the fingerprint is 
falsely matching at a much higher rate than 1/128 and forcing a check of almost every entry. This indicates poor 
hash mixing, and it seems likely AndroidX worst case performance could be improved with a stronger mixing function.

The more classic linear probing implementations (Fastutil, HPPC, PrimitiveCollections) also see performance 
degradation, but interestingly performance improves dramatically until at 32K entries before falling off again.

Agrona comes ahead as a winner in part to the extra mixing its hash function achieves.

#### Int → Int / getMiss

{{< benchmark-chart benchmark="IntMap.getMiss" order="random" title="Int → Int / getMiss — random keys" >}}

Eclipse (thanks to its 50% load factor) and AndroidX dominate getMiss performance. This is exactly what we'd expect 
from AndroidX as well - getMiss means the entire probe chain must be scanned before giving up, and AndroidX can scan 
long chains quicker than any other implementation thanks to its small (and thus easy to keep in cache) metadata 
array and SWAR techniques to scan 8 slots at a time. Note also however the performance falling of a cliff at 32M+ 
entries (benchmarks at larger sizes timed out completely). I'm unsure exactly what causes this, though I wonder if it's 
related to the metadata array no longer fitting in L3 cache + AndroidX using cache unfriendly geometric probing?

{{< benchmark-chart benchmark="IntMap.getMiss" order="lowBits" title="Int → Int / getMiss — lowBits keys" >}}

As we've learned to expect, Trove is more competitive with lowBits keys.

{{< benchmark-chart benchmark="IntMap.getMiss" order="highBits" title="Int → Int / getMiss — highBits keys" >}}

AndroidX again is particularly vulnerable against highBits keys, and still has the performance cliff at 32M+. 
Fastutil, HPPC and PrimitiveCollections all show the exact same odd spike in performance at ~95K elements - I am at 
a loss as to what might be causing that.

One interesting thing to note is I believe (though I have not confirmed) that we can see the effect of the JRE 
fallback to red-black trees instead of linked lists for storage - note how JRE performance recovers at ~10M elements.

#### Int → Int / putHit

{{< benchmark-chart benchmark="IntMap.putHit" order="random" title="Int → Int / putHit — random keys" >}}

Our "usual suspects" for linear probing implementations appears to be more competitive with Eclipse for put 
operations rather than get operations. Otherwise, nothing to shocking here, though JRE performance does appear to 
fall off a cliff at large sizes...

{{< benchmark-chart benchmark="IntMap.putHit" order="lowBits" title="Int → Int / putHit — lowBits keys" >}}

As before, Trove continues to perform better for lowBits keys. Nothing too shocking otherwise.

{{< benchmark-chart benchmark="IntMap.putHit" order="highBits" title="Int → Int / putHit — highBits keys" >}}

As before, several hashtables are flummoxed by highBits keys, and Agrona and Eclipse seem to have the most stable 
performance.

#### Int → Int / putMiss

{{< benchmark-chart benchmark="IntMap.putMiss" order="random" title="Int → Int / putMiss — random keys" >}}

As in the putHit benchmark, the linear probing implementations remain competitive with Eclipse. I am surprised 
AndroidX didn't perform better - as putMiss should require scanning to the end of a probe chain (which AndroidX is 
exceptionally fast at), but perhaps the additional write overhead in AndroidX as it twiddles bits is offsetting this?

{{< benchmark-chart benchmark="IntMap.putMiss" order="lowBits" title="Int → Int / putMiss — lowBits keys" >}}

Not much new to observe here.

{{< benchmark-chart benchmark="IntMap.putMiss" order="highBits" title="Int → Int / putMiss — highBits keys" >}}

Nor here - same patterns we observed previously persist.

#### Int → Int / removeAndPutMiss

{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="random" title="Int → Int / removeAndPutMiss — random keys" >}}

Almost all the implementations demonstrate a more pronounced sawtooth here, as they become more sensitive to the exact
load factor. The JRE HashMap performs quite well, beating most other unboxed implementations, including Eclipse, 
until it gets to ~512K elements. AndroidX performance falls off a cliff with large numbers of elements, and this 
does not appear to be related to the keys at all, since this performance cliff is visible for both lowBits and 
highBits removeAndPutMiss benchmarks as well.

{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="lowBits" title="Int → Int / removeAndPutMiss — lowBits keys" >}}

Overall not much change for lowBits benchmarks vs random - do note however that Trove has *not* improved relative 
to random, as we saw for all the get/put benchmarks...

{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="highBits" title="Int → Int / removeAndPutMiss — highBits keys" >}}

Largely the same patterns we've seen before here.

#### Int → Int / forEach

{{< benchmark-chart benchmark="IntMap.forEach" title="Int → Int / forEach" >}}

The time it takes to iterate over a hashtable is heavily influenced by the cost of skipping empty entries.

#### Int → Int / naiveCopy + preAllocatedCopy

{{< benchmark-chart benchmark="IntMap.naiveCopy" title="Int → Int / naiveCopy" >}}
{{< benchmark-chart benchmark="IntMap.preAllocatedCopy" title="Int → Int / preAllocatedCopy" >}}

### Long → Int

Now the results of benchmarking Long → Int hashtables. Note that since Agrona does not support Long → Int tables, it 
is being benchmarked with a Long → Long table.

#### Long → Int / getHit

{{< benchmark-chart benchmark="LongMap.getHit" order="random" title="Long → Int / getHit — random keys" >}}
{{< benchmark-chart benchmark="LongMap.getHit" order="lowBits" title="Long → Int / getHit — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.getHit" order="highBits" title="Long → Int / getHit — highBits keys" >}}

#### Long → Int / getMiss

{{< benchmark-chart benchmark="LongMap.getMiss" order="random" title="Long → Int / getMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.getMiss" order="lowBits" title="Long → Int / getMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.getMiss" order="highBits" title="Long → Int / getMiss — highBits keys" >}}

#### Long → Int / putHit

{{< benchmark-chart benchmark="LongMap.putHit" order="random" title="Long → Int / putHit — random keys" >}}
{{< benchmark-chart benchmark="LongMap.putHit" order="lowBits" title="Long → Int / putHit — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.putHit" order="highBits" title="Long → Int / putHit — highBits keys" >}}

#### Long → Int / putMiss

{{< benchmark-chart benchmark="LongMap.putMiss" order="random" title="Long → Int / putMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.putMiss" order="lowBits" title="Long → Int / putMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.putMiss" order="highBits" title="Long → Int / putMiss — highBits keys" >}}

#### Long → Int / removeAndPutMiss

{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="random" title="Long → Int / removeAndPutMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="lowBits" title="Long → Int / removeAndPutMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="highBits" title="Long → Int / removeAndPutMiss — highBits keys" >}}

#### Long → Int / forEach

{{< benchmark-chart benchmark="LongMap.forEach" title="Long → Int / forEach" >}}

#### Long → Int / naiveCopy + preAllocatedCopy

{{< benchmark-chart benchmark="LongMap.naiveCopy" title="Long → Int / naiveCopy" >}}
{{< benchmark-chart benchmark="LongMap.preAllocatedCopy" title="Long → Int / preAllocatedCopy" >}}
