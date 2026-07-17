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
or that the default JRE implementation is all-around quite reasonable - efficient, resistant against attacks, etc... 
However, I'm interested in focusing entirely on performance right now.

For today, we're interested specifically in hashtables for primitive values. By focusing on values rather than
references we'll entirely ignore the cost of heap allocation, pointer dereferencing, GC pressure, and also come up 
with an analysis that will hopefully continue to hold weight with upcoming
[Valhalla](https://openjdk.org/projects/valhalla/) changes to the JVM.

Hashtables that store values rather than references can provide a range of large performance benefits within the JVM,
such as no boxing, no pointer indirection, better cache locality, etc... The JVM ecosystem has accumulated a handful of
libraries that address this by providing hashtables backed by primitive arrays. The problem is that the performance 
tradeoffs and design choices between them are neither obvious nor well-documented anywhere. Many different choices 
are made around hash functions, collision resolution strategies, load factors, and those choices produce
meaningfully different performance profiles yet there is pretty much no data available on head-to-head performance.

We'll benchmark and analyze the following libraries (links go to more detailed information lower down this page):

* [JRE](#jre)
* [FastCollect](#fastcollect) (I am the author of FastCollect)
* [Fastutil](#fastutil)
* [AndroidX](#androidx)
* [Trove](#trove)
* [Koloboke](#koloboke)
* [Eclipse](#eclipse)
* [HPPC](#hppc)
* [Agrona](#agrona)
* [PrimitiveCollections](#primitivecollections)

You may note that several of the libraries benchmarked here are quite old and/or no longer maintained. Others are 
not intended for general purpose usage. The larger purpose of this post is not to determine which is fastest, but to 
investigate the design choices that affect hashtable performance. Also note that the JRE is the only library here 
truly suited for external use (when an attacker can control the table) - most of these libraries have abandoned any 
pretense at resisting DoS attacks in favor of raw performance.

Full benchmarking code is available at https://github.com/sooniln/jvm-collections-benchmarks.

## Hashtable Design Choices

Before we begin, it's worth starting with a quick refresher on how hash tables work, and the various design choices that
have sprung up over the years. If you're already reasonably familiar with how hashtables work, feel free to skip 
straight to the [Libraries Under Test](#libraries-under-test) section, or the [Benchmark Setup](#benchmark-setup).

Pretty much all general purpose hashtables operate by mapping a very large universe of potential keys onto a much
smaller number of available slots (represented by an in-memory data structure, almost always an array). As a trivial
example, if we're using 32-bit integers as keys, we would need to map all the ~4 billion possible integers onto the much
smaller array in memory (for example a 128 slot array perhaps, if we're only expecting to map 100 integers total). This
is accomplished via [hash functions](https://en.wikipedia.org/wiki/Hash_function).

Now by definition, once we've computed such a mapping there may be collisions - two different keys that are mapped 
by a hash function to the same slot. The second responsibility of the hashtable is thus to handle collisions, and
store multiple keys that map to the same slots. This brings us to our first design decision,
[open addressing](https://en.wikipedia.org/wiki/Open_addressing) vs
[separate chaining](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining).

### Open Addressing vs. Separate Chaining

The first architectural fork in hashtable design is how collisions are handled when two keys map to the same slot.

**Separate chaining** associates a separate collection of entries with each slot. All colliding entries thus are stored 
within the same collection. The advantage here is simplicity - it's trivial to add and remove things, and the selection
of different collection types modifies performance. The most common collection used here is a linked list, but other
options are possible (the JRE HashMap starts with a linked list for example, but converts to a red-black tree if the
list grows too large). The downside is an additional indirection to enter the collection, and the cost of lookup within
the collection (pointer traversal such as in a linked list is very cache-unfriendly for example).

**Open addressing** stores all entries directly in the contiguous backing array. When a slot is occupied, the algorithm
probes for the next candidate slot according to some probing strategy (discussed in more detail below). Lookups and 
insertions are likely to stay near the original slot (cache-friendly), which is why open addressing dominates value 
hashtables (which don't have the built-in pointer dereferencing cost of references). Deletion in an open-addressed 
table requires extra thought, and is discussed in more detail below.

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

At α=0.5, this means ~1.5 probes for hits and ~2.5 for misses. At α=0.75 that's ~2.5 probes for hits and ~8.5 for 
misses. And at α=0.9: ~5.5 for hits, ~50.5 for misses! This is a very simple calculation, assuming a perfect hash 
function and the simplest possible version of linear probing, but it is useful for understanding general probe 
length behavior.

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
advantage to prime sizing - for non-linear probing schemes such as quadratic probing (discussed later), it is much 
easier to ensure no infinite loops in probing (every slot will be visited by the probe before repeating).

The simple implementations of both of these schemes require a large re-allocation and complete rehash (re-insertion 
of all elements) whenever the hashtable runs out of room. There are scenarios where this large "pause the world" 
type of logic may be unacceptable, and so there are also
[variations](https://en.wikipedia.org/wiki/Hash_table#Alternatives_to_all-at-once_rehashing) that allow for more 
gradual resizing over time - usually at the expense of performance. All hashtables benchmarked here do complete rehashes
on running out of space (some allow for tombstone-removal-only rehashes in the event that they use tombstones - 
covered later).

### Hashtable Memory Layout

Hashtables need to store both keys and values. There are generally two simple memory layouts for keys and values:

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
memory - no extra load necessary. But while probing, every cache line can now hold fewer keys, and thus more cache
misses are incurred. Which layout is faster comes down to how much probing is required to find a key or ensure it is 
not present, and how many caches misses probing produces.

#### Metadata

Rather than storing keys/values directly in the primary hashtable arrays, there are other possibilities as well:

For example, [Robin Hood](https://en.wikipedia.org/wiki/Robin_Hood_hashing) hashtables may store probe sequence 
lengths (PSLs - how far a key is located from the slot it hashed to), and
[Swiss tables](https://abseil.io/about/design/swisstables) store key fingerprints. Given that the metadata is 
usually substantially smaller than the key size, metadata often cannot easily be interleaved with keys. If probing now 
requires accessing both metadata and keys this will incur many more cache misses, so when metadata is used, probing 
will almost always attempt to only access metadata (which is what makes key fingerprints so useful), and access keys 
only when absolutely necessary.

There are also hashtables that store indirect indices which point into separate key/value storage. This allows 
key/value storage to be much denser than the main hashtable, potentially saving space, at the cost of an additional 
level of indirection. This might allow a hashtable to use a load factor of 50% or lower, knowing that it will only 
apply to a small metadata table, while keys/values are packed tightly. None of the hashtables being benchmarked use 
this technique.

Techniques that use larger amounts of metadata can payoff quite well - but they tend to be used in more general purpose
hashtables which allow for storage of larger keys and values. When speaking of JVM primitives, which max out at 64 
bits, 32 bits of overhead is quite substantial...

Finally, there is one more advantage to metadata - it can help with representing empty slots, which will be discussed 
in the next section.

### Empty Slots

We've discussed key arrays, value arrays, interleaved key/value arrays, and separate metadata arrays. No matter which
is used however, a hashtable needs some way to indicate an empty slot, so that it knows where it can insert new 
entries or when it can stop probing during a lookup. The problem, of course, is that there is no guarantee any  
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

There are of course potential performance implications to all of these choices. If you need to add special cases,
you'll add to code size - too large of a code block can prevent some JVM C2 compiler optimizations and also incur 
more code cache misses. If you need to check a member variable this is an additional load and register usage. Too 
much register usage can cause register spilling and commensurate performance impacts on the hot path. No matter the 
choice there's always a variety of consequences to think through, and pretty much no way to anticipate the impact 
except through benchmarking.

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
that swaps entry positions (swapping a "poorer" \[further from its home slot\] entry with a "richer" \[closer to 
its home slot\] entry - hence Robin Hood) during insertion to reduce probe length variance. Entries that are "rich" 
and landed near their ideal slot yield their position to "poor" entries that have traveled further. This equalizes 
probe lengths and allows much higher load factors without the worst-case blow-up of plain linear probing (Robin Hood 
hashtables are often run at load factors of 80%+ without much performance impact).

And finally of course, you can have arbitrarily complex combinations of any probing scheme. A table could use linear 
probing for the first N elements, then switch to quadratic probing, or switch to linear probing with a different hash 
function, etc... The advantage to more complex probing schemes is that you can attempt to equalize the various pros 
and cons of each. For example, by starting with linear probing you can take advantage of its cache friendly behavior 
initially - linear probing for a cache line distance to avoid extra cache misses - then switch to quadratic probing 
when you're going to incur a cache miss anyway. The downside of these arbitrarily complex combinations is of course 
code complexity - but also entry removal, which can become very difficult.

### Entry Removal

Now that we've walked through the process of adding entries to the hashtable, how do we remove entries? Deleting a key 
from an open-addressed map is more complex than insertion. We can't simply find the slot in the array and clear 
it - this would break the invariants various probing algorithms rely on to function. Clearing the slot would 
leave a gap in a probe sequence, causing lookups for later keys to terminate early at the gap and incorrectly report 
a key as not present in the table.

**[Lazy deletion](https://en.wikipedia.org/wiki/Lazy_deletion)** is one of the more common implementations - rather 
than clearing the slot, mark it with a special DELETED value (called a tombstone). Future lookups probe past 
tombstones, and future insertions can reuse them. Tombstones are very easy to implement, but can accumulate over 
time. A map with many insertions and deletions will fill up with tombstones, which probe sequences must scan through 
even though they represent no data. This degrades all operations over time. Most implementations address this by 
triggering a rehash when tombstones exceed some fraction of capacity, but this means delete-heavy workloads 
periodically pay an additional full-table rehash cost.

**Backwards-shift deletion** is an alternative that avoids tombstones entirely. After clearing a slot, the algorithm 
looks at the next slot and checks if its occupant could shift back one probe position (i.e., its probe distance is 
greater than zero — it's not at its ideal slot). If so, it shifts back. This continues until an empty slot or a slot 
whose occupant is already at its ideal position is reached.

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
(1) element displacement must be linear (2) an element's displacement must be calculable - not every insertion 
strategy makes it possible to deterministically know an element's displacement. Robin Hood hashing tends to maintain 
this naturally, so backwards-shift deletion is commonly used with Robin Hood hashing, but it is possible with other 
variations as well.

### Hash Functions And Mixing: Avalanche and Bias

So far we've discussed the simple mechanics of how a hashtable works, but avoided thinking too much about the hash 
function (often assuming a uniformly random hash function).

Theoretically, the ideal is a **perfect hash function**. This is a hash function that maps a set of keys to a set of 
slots with no collisions at all. This is generally only achievable if the complete set of keys is known in advance 
(which is not realistic for general-purpose hashtables, but may be in subdomains such as compilers which need to map 
keywords, etc...). There are a variety of algorithms for attempting to construct perfect hash functions, but as we are 
concerned with general purpose tables we will not delve deeper or more greedily into the subject.

For general purpose scenarios, the ideal is a **uniformly random hash function**. Uniformly random hash functions are a
theoretical construct for analysis where the collision probability of any two distinct keys is 1/N (where N is the 
number of slots). This is essentially the idealized limit that real hash functions try to approximate.

Real hash functions can be evaluated on several axes:

* **Collision rate**: How often distinct keys are mapped to the same slot.
* **Uniformity**: How evenly outputs are spread across the range of slots.
* **Independence**: Whether the slots chosen for related keys exhibit statistical independence.
* **Throughput**: Hashes/Second on bulk input.
* **Latency**: Cycles/Hash for a single key.
* **Attack resistance**: Can a seed be provided so mapping is unpredictable to an attacker? Is cryptographic 
  strength required or unnecessary?

In order to construct hash functions with good uniformity and independence, we often use the terms:

**Avalanching**: Sensitivity to input changes - the ideal is that flipping a single bit in the input should cause 
50% of the output bits to flip, and the bits that flip are effectively randomly chosen.
**Bias**: Do any particular outputs or output patterns appear far often than they should (favoring some slots over 
others and thus increasing collisions)?
**Entropy**: Another way of discussing avalanching and bias (avalanching means each output bit should have an 
entropy close to 1 bit conditionally on a single input bit flip, bias means for N bit output, ideal entropy is N 
bits) - essentially just reframing these in information-theoretic terms.

It's important to remember however that these methods of evaluating the strength of a hash function are generally 
predicated on the assumption that keys will be drawn from random input. If there are stronger guarantees on the key 
space, then perhaps not all of these properties are useful. A common example of this, what if we can assume integer 
keys can be mapped to [0-N) where N is the table size? This is not an uncommon assumption in many domains, and in 
this case the identity hash function (i → i) is actually a perfect hash function - even though the identity hash 
exhibits pretty much no independence (linear key relationships are preserved) and awful avalanching (flipping 1 bit of 
input = 1 bit changed in output).

#### Knuth Multiplicative Hashing

One of the most common choices of hash function is Knuth Multiplicative Hashing (introduced by Donald Knuth in TAOCP).
The idea is simply to multiply the input by a large "irrational-ish" constant and extract the high-order bits.

**$h(k) = \left\lfloor m \cdot \left( kA \bmod 1 \right) \right\rfloor$**

* A: A fractional constant between 0 and 1. Knuth proved that the ideal distribution is achieved when
  A ≈ $(\sqrt{5}- 1) / 2$, which is the inverse of the golden ratio (φ⁻¹). Moving forward we'll refer to φ as PHI 
  for simplicity.
* m: The number of slots to map.

When we use the specific inverse PHI (φ⁻¹) constant Knuth suggested for its maximally irrational properties, this is 
known as Fibonacci hashing.

Variants on Knuth multiplicative hashing are used by the FastCollect, Fastutil, AndroidX, Koloboke, HPPC, 
and PrimitiveCollections libraries. Knuth multiplicative hashing by itself does not have great avalanching 
properties and tends to concentrate entropy in higher bits (rather, the entropy is in the fractional part of the 
result, which since we are using integer multiplication and PHI values adjusted by 2^32, ends up being the high 
bits of the result), so common variants add a xor-shift operation to (1) improve avalanching (2) improve entropy in 
the lower bits (recall power-of-two sized tables use `hash & (n-1)` to mask out the slot):

```java
final int INT_PHI = 0x9E3779B9 // 2^32 / PHI, where PHI is the golden ratio

int hash(int k) {
    int h = k * INT_PHI;
    return h ^ (h >>> 16); // fold high bits into low bits
}
```

Knuth multiplicative hashing is only one hashing strategy however, and there exist many other possibilities, all 
with their own lists of upsides and downsides.

## Libraries Under Test

### JRE

Java's standard library hashtable uses separate chaining: each bucket holds a linked list that is automatically 
promoted to a red-black tree once it grows too large, bounding worst-case lookup from O(n) to O(log n) per bucket 
under adversarial key distributions. All keys are stored as boxed primitive objects on the heap; every key thus incurs
an allocation and a pointer indirection on access.

* **Storage Schema**: Power-of-two sized, array of node pointers holding boxed keys/values (can form a linked list or 
  red-black tree)
* **Probing Strategy**: Separate chaining (linked list/red-black tree)
* **Deletion Strategy**: Removal from linked list/red-black tree
* **Default Load Factor**: 75%

**Integer key hash finalizer**: The hash finalizer performs minimal mixing, relying on the red-black tree fallback 
behavior to prevent pathological lookups.

```java
return key ^ (key >>> 16)
```

### FastCollect

*Disclosure: I am the author of FastCollect.*

FastCollect is a Kotlin Multiplatform primitive collections library supporting JVM, JS, and native targets. The map 
implementation currently uses Robin Hood hashtables because I wanted to play around with them.

**Source**: [sooniln/fastcollect](https://github.com/sooniln/fastcollect)  
**Maven**: `io.github.sooniln:fastcollect-kotlin-jvm:2.0.2`  
**Last release**: June 2026

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear probing (Robin Hood)
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 83.3% (100% for very small tables)

**Integer key hash finalizer**: FastCollect takes a different approach to multiplicative hashing than most of the 
other libraries benchmarked. Several of the other libraries all use the same hash finalizer (which appears to have 
originally come from the Koloboke library many years ago - `(h * PHI) ^ ((h * PHI) >>> 16)`). It seems worth 
revisiting this finalizer as it exhibits several poor qualities that can be improved upon. While it is cheap, the 
multiplication concentrates entropy in the high bits, yet it is only the low bits that will be used to derive the 
slot. Some effort is made to ameliorate this with the xor-shift, but it is generally insufficient. This finalizer 
exhibits worse results when given non-random inputs.

For the FastCollect finalizer the key realization is that simply reversing the order of bits in the multiplication 
result puts the most entropy exactly where we want it, in the low bits. The problem is that while bit-reversal is a 
single instruction (RBIT) on ARM, x86-64 does not have a similar instruction. Divide and conquer + BSWAP approaches 
and similar can make bit-reversal reasonably cheap, but not single cycle cheap like RBIT. Since benchmarking is 
occurring on x86-64 we choose a slightly different approach: premix bits, multiply by PHI, and then reverse the 
bytes (not bits) with BSWAP/REV.

```java
var h = (key ^ (key >>> 16)) * 0x9E3779B9  // PHI
return Integer.reverseBytes(h)
```

It would be very interesting to look into RBIT finalizers more on ARM platforms, it seems a much cheaper way of 
achieving higher levels of entropy in a targeted part of the bit pattern, rather than focusing on achieving even 
levels of entropy in all parts of the pattern (as more complex functions like murmur3, etc... appear to do). I am not 
aware of any hashtable implementations that have experimented with this, and it could be there are hidden weaknesses 
to this approach that I am not aware of.

### Fastutil

One of the most comprehensive and widely-adopted JVM primitive collections libraries, Fastutil is very well known.

**Source**: [vigna/fastutil](https://github.com/vigna/fastutil)  
**Maven**: `it.unimi.dsi:fastutil:8.5.18`  
**Last release**: October 2025

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9
return h ^ (h >>> 16)
```

### AndroidX

Google's Jetpack collection utilities library, now written in Kotlin and available as a Kotlin Multiplatform artifact.
The implementation is a Swiss Table design that uses SWAR (SIMD within a register) techniques to evaluate groups
of 8 entries at a time. AndroidX libraries are primarily intended for arm64, though they are distributed as 
multi-platform. Since benchmarks were run on amd64, care should be taken not to generalize the results.

**Source**: [android.googlesource.com](https://android.googlesource.com/platform/frameworks/support) ([GitHub mirror](https://github.com/androidx/androidx))  
**Website**: [developer.android.com/jetpack/androidx/releases/collection](https://developer.android.com/jetpack/androidx/releases/collection)  
**Maven**: `androidx.collection:collection-jvm:1.6.0`  
**Last release**: March 2026

* **Storage Schema**: Power-of-two sized, metadata array of 1 byte/entry, parallel arrays for keys/values
  * **Metadata Schema**: Stores low 7 bits of original hash as the metadata fingerprint, the high 25 bits are used to
    drive the probe offset
* **Probing Strategy**: Quadratic Probing (where each probe checks a group of 8 linear entries)
* **Removal Strategy**: Tombstones + rehash to remove tombstones at ~78% capacity
* **Default Load Factor**: 87.5%

**Integer key hash finalizer**: Murmur3 finalizer. Note that the left shift instead of right shift would seem to
concentrate hash entropy further in the upper bits. This makes some sense for the Swiss Table design, where the 
initial probe position is calculated from the high bits and the fingerprint is derived from the low bits. But this 
opens up the risk of perhaps too little entropy in the low bits and thus the fingerprint (see benchmarking results for 
highBits order)...

```java
var h = key * 0xCC9E2D51 // MurmurHashC1
return h ^ (h << 16)
```

### Trove

GNU Trove uses prime sizing and open addressing with double hashing: rather than a fixed linear step, each key 
computes a secondary probe stride from its primary hash, so colliding keys scatter along independent paths and form 
shorter clusters than linear probing would.

**Source**: [bitbucket.org/trove4j/trove](https://bitbucket.org/trove4j/trove)  
**Maven**: `net.sf.trove4j:core:3.1.0`  
**Last release**: August 2018 (due to module split - no active development since ~2012 apparently)

* **Storage Schema**: Prime sized, metadata array to track slot empty state + parallel arrays for keys/values
    * **Metadata Schema**: Only tracks empty state, which means every probe step has to check both metadata array and
      key arrays - would expect this to hurt performance.
* **Probing Strategy**: Double hashing
* **Removal Strategy**: Tombstones, rehash if there are no free slots (only tombstones) left, also rehash if removal 
  allows backing array to shrink to the previous prime size
* **Default Load Factor**: 50%

**Integer key hash finalizer**: The identity hash — no bit mixing is applied; distribution quality relies entirely on
the double-hashing probe and prime sized arrays to counteract clustering.

```java
// double hashing
// slot - identity hash
return key

// stride
return 1 + (key % (length - 2))
```

### Koloboke

A high performance collections library from Roman Leventov, appears to have been designed with HFT in mind. It makes
use of deprecated Unsafe APIs for performance.

**Source**: [leventov/Koloboke](https://github.com/leventov/Koloboke)
**Maven**: `com.koloboke:koloboke-impl-jdk8:1.0.0`  
**Last release**: May 2016

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 66.7%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

### Eclipse

Originally the Goldman Sachs Collections library, donated to the Eclipse Foundation in 2012. Eclipse forces a load 
factor of 50% - much lower than any other library in this benchmark - and it cannot be adjusted. Beware assuming 
an apples to apples comparison. Some experimentation with running other libraries with a 50% load factor indicates 
they are competitive and may outperform Eclipse, but full benchmarking has not been performed.

**Source**: [eclipse/eclipse-collections](https://github.com/eclipse/eclipse-collections)  
**Website**: [eclipse.dev/collections](https://eclipse.dev/collections/)  
**Maven**: `org.eclipse.collections:eclipse-collections:13.0.0`  
**Last release**: December 24, 2024

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Combination (linear probing on hash1 for half a cache line → linear probing on hash2 for 
  half a cache line → double hashing)
* **Removal Strategy**: Tombstones - tombstones count towards rehashing load factor
* **Default Load Factor**: 50%

**Integer key hash finalizer**: 3 separate hash finalizers for the various probing steps. It's not entirely clear where 
the constants come from.

```java
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

High Performance Primitive Collections, maintained by Carrot Search.

**Source**: [carrotsearch/hppc](https://github.com/carrotsearch/hppc)  
**Website**: [labs.carrotsearch.com/hppc.html](https://labs.carrotsearch.com/hppc.html)  
**Maven**: `com.carrotsearch:hppc:0.10.0`  
**Last release:** June 2024

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

### Agrona

Agrona is Real Logic's utility library, developed primarily to support the [Aeron](https://github.com/aeron-io/aeron)
messaging system. Agrona requires that every map must have an unrepresentable value passed in on initialization (to 
mark empty slots); thus Agrona is incapable of representing the full range of values.

Note: Since Agrona does not have a Long → Int map, benchmarking uses Long → Long maps instead, and widens values. Be 
wary of apples to apples comparisons.

**Source**: [aeron-io/agrona](https://github.com/aeron-io/agrona)
**Maven**: `org.agrona:agrona:2.4.1`  
**Last release**: April 2026

* **Storage Schema**: Power-of-two sized, interleaved arrays for keys/values (Agrona does not support keys and values 
  with different sizes)
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 65%

**Integer key hash finalizer**: Two-round XOR-shift/multiply from Chris Wellons' Hash Prospector.

```java
var h = key ^ (key >>> 16)
h = h * 0x119DE1F3
h = h ^ (h >>> 16)
h = h * 0x119DE1F3
return h ^ (h >>> 16)
```

### PrimitiveCollections

PrimitiveCollections is a library explicitly aimed at duplicating and outperforming Fastutil.

**Source**: [Speiger/Primitive-Collections](https://github.com/Speiger/Primitive-Collections)  
**Maven**: `io.github.speiger:Primitive-Collections:1.0.0`  
**Last release**: May 2026

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9 // PHI
return h ^ (h >>> 16)
```

## Benchmark Setup

* No CPU pinning, ran on idle machine.
* 6 Core AMD Ryzen 5 9600X - Windows 11
* L1/2/3 Cache Sizes: 48KB/core, 1MB/core, 32MB shared

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
similar comparisons we try to enforce that every hashtable has a load factor of at least 75% (higher is allowed). 
Eclipse does not allow for load factor adjustments, and remains at 50%.

| Library              | Default | Benchmarked | Adjustable |
|----------------------|---------|-------------|------------|
| JRE                  | 75%     | 75%         | Yes        |
| FastCollect          | 83.3%   | 83.3%       | No         |
| Fastutil             | 75%     | 75%         | Yes        |
| AndroidX             | 87.5%   | 87.5%       | No         |
| Trove                | 50%     | 75%         | Yes        |
| Koloboke             | 66.7%   | 75%         | Yes        |
| **Eclipse**          | 50%     | **50%**     | No         |
| HPPC                 | 75%     | 75%         | Yes        |
| Agrona               | 65%     | 75%         | Yes        |
| PrimitiveCollections | 75%     | 75%         | Yes        |

### Key Selection and Ordering

Because hash functions have such a large effect on the performance of hash tables, it's important to test with data 
that can represent real world scenarios. We specify 3 key orders to test under:

* **random**: Keys are selected at random from the entire universe of possible keys. This represents scenarios where
  only a subset of keys are stored - for example if a hashtable is being used as a cache.
* **lowBits**: Keys are selected from 0 → table size and shuffled randomly. This has the effect that only the least 
  significant bits of the key tend to be set, and most significant bits are all zero. This represents scenarios where 
  keys are generally linear and almost all keys are stored - for example internal id values.
* **highBits**: The same as lowBits, but every key is bit-reversed, so that the most significant bits tend to be set,
  and the least significant are all zero. This represents two scenarios - (1) an easy adversarial attack (since the 
  vast majority of hashtables are power-of-two sized, we know that they will usually truncate high bits in order to 
  calculate a slot - without good bit-mixing properties these tables may be vulnerable) (2) structures that have been 
  bit-packed into keys and thus have unusually structured bits (for example, a Point class can trivially be encoded as 
  a 64-bit value).

In practice, I suspect (with no evidence other than gut feeling) that random and lowBits are likely to represent the 
majority of real-world scenarios in the JVM ecosystem.

### Benchmark Selection

We benchmark the following operations:

* **GetHit**: The cost of looking up a value for a key present in the hashtable. Note that since different 
  hashtables have different conventions for representing keys that are not present (some of which may give that 
  hashtable an unfair advantage), we use getOrDefault() or equivalent APIs for a fair comparison.
* **GetMiss**: The cost of looking up a value for a key not present in the hashtable. Note that since different
  hashtables have different conventions for representing keys that are not present (some of which may give that
  hashtable an unfair advantage), we use getOrDefault() or equivalent APIs for a fair comparison.
* **PutHit**: The cost of changing the value for a key present in the hashtable. When offered, this will always use 
  an API which does not return a value in order to find the cheapest possible cost.
* **PutMiss**: The cost of assigning a value to a key not present in the hashtable. When offered, this will always use
  an API which does not return a value in order to find the cheapest possible cost.
* **RemoveAndPutMiss**: The cost of removing a key and then assigning a value to a different key that is not present in
  the table. The setup of JMH tests makes it difficult to accurately quantify the cost of just a remove operation - 
  this is a compromise.
* **ForEach**: The cost of iterating over every key/value present in the hashtable using the fastest iteration 
  technique exposed by the hashtable (likely not iterators).
* **NaiveCopy**: The cost of copying the hashtable starting from empty and inserting elements until complete - 
  with no information given on the total number of elements to be expected, so no pre-allocation can be done.
* **PreAllocatedCopy**: The cost of copying the hashtable directly from another hashtable (so size and layout is 
  known - may just be a memcpy in the cheapest case).

### Micro-Benchmarking Caveats

Micro-benchmarking (as we're performing here) does not give you real world performance results. In the best case, it 
can give you a "platonic ideal" of performance for primitive operations (which real world results can approach, but 
will likely never achieve). These methods are being benchmarked in pretty much the most optimal conditions possible, 
branches are highly predictable, memory access patterns are similar every run, etc... It is difficult to overstate 
how much this can affect performance.

The other major problem is that micro-benchmarking results are not composable: If `a()` is benchmarked at 5ns, and
`b()` is benchmarked at 10ns, is `a() + b()` = 15ns? Absolutely not - microbenchmarking results fall apart when
you try to compose them. The benchmark of X.a() might be run with the precondition that condition Y is always true, and
the benchmark of X.b() might be run with the precondition that condition Y is always false... The trivial example is 
getHit() and getMiss() benchmarking in our results. The precondition of getHit() is that the lookup key is always in 
the table. The precondition of getMiss() is that the lookup key is never in the table. Benchmarking for both can achieve
almost perfect branch prediction because of this. Real world performance is going to be highly dependent on the 
exact pattern of hits/misses real world data gives you.

In addition to the standard micro-benchmarking caveats, there are also hash table specific caveats. Notably, 
hashtables that use tombstones for removal can have the tombstones substantially affect lookup times - but we 
performed no benchmarking of removal + lookup scenarios. There are many other scenarios that could affect performance
that were also uncovered in any benchmarking - it's impossible to cover everything.

## Benchmark Results

Raw benchmark results can be downloaded [here](/libraries.csv).

While viewing graphs, you may use the button to switch the time axis between log scale and linear scale. Clicking on 
a library in the legend will hide/show the data for that library. The mouse can be used to pan/zoom the graphs. 
Double-clicking a graph will reset its pan/zoom.

### Overall Results

Rather than forcing you to scroll past the excessive graphs and numbers below which go into individual scenarios, 
might as well cut to the chase, right?

For map iteration and copying benchmark results, find the relevant [All Results](#all-results) section.

#### Memory Usage

Memory usage is measured with the [JOL (Java Object Layout)](https://github.com/openjdk/jol) library. Note that when 
some libraries are missing data at higher sizes this is because they timed out. This is particularly relevant for 
JRE and AndroidX - both lack an ensureCapacity() API, and thus cannot be efficiently filled with elements in our 
memory benchmark. Note also that several libraries use pretty much the exact same amount of memory, and thus their 
data series are overlaid on top of one another in the graph. Recall that clicking a library in the legend will 
hide/show its data series.

{{< benchmark-chart benchmark="IntMap.memory" title="Int → Int Map Memory Usage" yAxis="linear" >}}

Results are pretty much as we'd expect - JRE maps which store objects take up 3-6.5x as much memory as maps which store
primitives. The effects of prime sizing (Trove) vs power-of-two sizing (everyone else) and different load factors 
are also on display.

{{< benchmark-chart benchmark="LongMap.memory" title="Long → Int Map Memory Usage" yAxis="linear" >}}

Results are pretty much the same, except that Agrona now uses more memory (recall that as a special-purpose library 
Agrona does not support Long → Int maps, and thus we use its Long → Long map instead).

#### Read Performance

Read performance is calculated as the geometric mean of GetHit and GetMiss benchmarks. This weights both hits and 
misses equally, an assumption which may not be true in various real world workloads. Results for the three orderings 
(random, lowBits, highBits) follow:

{{< benchmark-chart benchmark="Map.readGM" order="random" title="Map Read Geometric Mean — random keys" >}}

As this is the first graph we're looking at, note the distinct regime changes as the hashtable size increases past
L1/L2/L3 cache sizes. For many of the power-of-two sized graphs we see the distinct sawtooth performance that comes
from different load factors as the table is queried when more empty vs more full (the sizes benchmarked were 
explicitly chosen to differentiate this). Looking at AndroidX, we can note two things - (1) a more subdued change in 
performance as size increases, and (2) it hits a performance cliff and times out around 32M entries. Further 
analysis of AndroidX can be found later. We also note that JRE read performance is relatively competitive (although 
it uses far more memory). Eclipse is the obvious fastest across all sizes - however how much of its performance is 
due to the fact that it is using an unfairly (for comparison purposes) lower load factor? We'll dig into that later 
as well.

{{< benchmark-chart benchmark="Map.readGM" order="lowBits" title="Map Read Geometric Mean — lowBits keys" >}}

Recall that lowBits keys concentrate their entropy in the lower bits, which is where most of these tables calculate 
slot positions from. The biggest change we see here is with Trove performance (it was one of the slowest for random 
keys, but is now among the fastest for lowBits keys). As covered earlier, Trove uses the identity hash (which is 
"perfect" for lowBits keys), so it is unsurprising that it performs so well with them. Eclipse also improves its 
performance - Eclipse's first choice of hash finalizer is also the identity hash.

{{< benchmark-chart benchmark="Map.readGM" order="highBits" title="Map Read Geometric Mean — highBits keys" >}}

Recall that highBits keys concentrate their entropy in the higher bits, which most of these tables ignore in order 
to calculate slot positions. So we expect this to be an interesting "adversarial" order, and the results bear that 
out. AndroidX and HPPC in particular show pathological performance degradation, especially at smaller sizes, with 
AndroidX managing to take ~1600ns on average for a lookup in the absolute worst case. Fastutil and PrimitiveCollections 
also see performance taking a hit, though not to the same extreme factor. Interestingly, if we look at JRE 
performance, it's possible that we're able to see the effect of JRE's red-black tree fallback ameliorating the 
performance hit here (though I have not confirmed that). FastCollect performs sufficiently well with highBits keys 
that it is able to out-perform even Eclipse, though Eclipse is operating at a load factor of 50% and FastCollect at
83%.

#### Write Performance

Write performance is calculated as the geometric mean of the PutHit, PutMiss, and RemoveAndPutMiss benchmarks. This 
weights misses higher, as they are represented twice (PutMiss and RemoveAndPutMiss), an assumption which may not be 
true in various real world workloads. Results for the three orderings (random, lowBits, highBits) follow:

{{< benchmark-chart benchmark="Map.writeGM" order="random" title="Map Write Geometric Mean — random keys" >}}

We immediately see some patterns which look the same as from the read results, and note that AndroidX again hits a 
performance cliff and times out at higher sizes. JRE is less competitive for write than reads it would also appear 
here, and while Eclipse is again generally the fastest, it isn't as dominant as it was for reads (again with the 
caveat that Eclipse is operating at a different load factor than all the other libraries).

{{< benchmark-chart benchmark="Map.writeGM" order="lowBits" title="Map Write Geometric Mean — lowBits keys" >}}

Again, similar patterns for lowBits in writes from reads. Trove performs better with lowBits keys than random keys 
(thanks to its identity finalizer), and the same for Eclipse (with its first choice identity finalizer).

{{< benchmark-chart benchmark="Map.writeGM" order="highBits" title="Map Write Geometric Mean — highBits keys" >}}

And for highBits keys we see the same patterns for AndroidX and HPPC, awful performance, especially at lower sizes. 
Fastutil and PrimitiveCollections show the same degradation, just not as bad.

#### Takeaways

Overall primitive maps deliver a large amount of memory savings, and in some, but not all cases, a reasonable 
performance improvement as well. While JRE maps can be competitive for reads compared to some libraries, they do fall 
behind for writes. Whether to switch, and which library to use is likely entirely dependent on your real usage
patterns.

So far we haven't discussed iteration or copying at all, so let's take a look at two representative graphs from the 
[All Results](#all-results) section below. Let's start with iteration speed:

{{< benchmark-chart benchmark="IntMap.forEach" title="Int → Int / forEach" >}}

JRE iteration is obviously much slower than every other library - a direct consequence of how much more memory it 
uses. Accessing more memory in a linear fashion takes more time - no two ways about it. HPPC also has surprisingly 
slow iteration... This is a consequence of a design decision HPPC has made to prevent the user from shooting 
themselves in the foot with naive copy operations. Most of these maps iterate in the same order as elements appear 
in the backing array via their hashcode, which means iteration indirectly exposes hash order. Inserting elements in 
hash order is generally a worst case scenario, and can cause quadratic runtimes (but the fix is also generally 
trivial, simply pre-size the map appropriately). In order to prevent this, HPPC inserts a random step in iteration -
iteration no longer exposes hash ordering, but is also now not very cache friendly and thus is slower. Many of the 
other libraries suffer the same problem, but generally ignore it and allow the user to shoot themselves in the foot 
if they aren't careful with proper pre-sizing so that iteration is not artificially slowed.

FastCollect suffers the same problem as well, but takes a slightly different approach. The Robin Hood invariant 
maintained in the map means we can expect reasonable upper bounds on run length for various map sizes. If 
FastCollect detects pathologically long run lengths that should never be expected under any normal scenario, it will 
pre-emptively resize the map larger, thus pathologically quadratic cases.

Since we've just discussed naive copy scenarios in some detail, let's look instead at pre-allocated copy performance 
next:

{{< benchmark-chart benchmark="IntMap.preAllocatedCopy" title="Int → Int / preAllocatedCopy" >}}

There's perhaps not too much surprising here - copy performance is tied to write performance, except that 
FastCollect is able to substantially outperform all other maps. FastCollect attempts to use direct memcpy whenever 
possible in copying, and for the simple copy performed here (start with an empty map, copy all elements from another 
map, which is already sized correctly) this yields much faster performance than manual iteration and insertion.

Now that we've covered the basics, let's dig into a couple different areas of unusuality.

### Load Factor & Eclipse Performance

Eclipse generally out-performs all the other libraries we've benchmarked - but we've hypothesized that this is due
to the fact that it uses a load factor of 50%, where every other library uses a load factor of 75% or higher. Is
this actually true though? Or is the load factor not actually the defining factor in Eclipse's performance? We couldn't
perform full benchmarking (limited time), but here are the results for read geomean performance with all libraries
set to 50% load factor where possible (AndroidX excluded since load factor is not adjustable, and FastCollect using
a custom build).

{{< benchmark-chart benchmark="MapLF50.readGM" order="random" title="50% LF Map Read Geometric Mean — random keys" >}}

With all libraries at 50% load factor, Eclipse is now solidly middle of the pack for read performance, and
FastCollect is now the de-facto fastest by a small margin. Load factor was indeed the driving reason behind
Eclipse's performance.

### AndroidX Oddities

AndroidX benchmarks (1) appear to hit a performance cliff at > ~32 million entries (2) timeout at very high numbers 
of entries, so there is no data for > ~32 million entries (3) exhibit extremely poor performance with highBits keys.

#### Performance Cliff At ~32 Million

With about 32 million entries, AndroidX's metadata array (1 byte per entry) jumps from ~32MB to ~64MB, and thus no
longer fits in the L3 cache of the benchmarking machine (32MB). Almost every probe is hitting cold memory
(perhaps further exacerbated by the geometric probing sequence?). It's worth remembering that AndroidX is primarily
intended for use on Android devices, which are likely to have a much smaller L3 cache than the desktop device used
for this benchmarking, but also are unlikely to be dealing with such large maps.

#### Benchmark Timeout

The timeout which leads to no benchmark data is not occurring within the benchmark, but within the setup method for 
the benchmark, while the hashtable is being loaded. AndroidX does not offer any ensureCapacity() functionality, 
meaning we cannot pre-size the table appropriately to reduce the cost of loading it - loading must go through a full 
set of rehashes, which is much more expensive. There are a couple other tables that also do not offer ensureCapacity()
either however and do not time out, why is AndroidX still more expensive than these tables?

This comes down to a couple features of AndroidX's design:
* Three separate tables (metadata, keys, values) - more cache misses than 1-2 tables.
* Metadata table must be initialized with a non-zero value - 2x the initialization cost on this table.
* Higher load factor (87.5%) means more elements to rehash every rehash.

I suspect that if AndroidX offered an `ensureCapacity()` API it would ameliorate these issues sufficiently to allow for
benchmarking at higher sizes and further investigate the performance cliff. Note that AndroidX does allow pre-sizing 
via the constructor - and we technically could have made use of this in the benchmark. However, that would have 
necessitated a different benchmarking structure for AndroidX than for every other library, and wasn't worth the
trouble.

#### Performance Drop For highBits Keys

In order to understand why AndroidX is so much more affected by highBits keys we'll need to dig into how exactly the 
library handles storage in the first place. AndroidX primarily uses a metadata array for lookups, with an equivalent 
key and value array. We'll focus only on the metadata array for now. When a key is hashed, the resulting 32-bit hash 
is separated into two parts:

* H1: the hash's 25 most significant bits - used to determine the initial slot via masking by table size
* H2: the hash's 7 least significant bits - stored as a fingerprint within the metadata array

Let's examine what happens with a highBits example:

```
int hash(int key) {
  var h = key * 0xCC9E2D51 // MurmurHashC1
  return h ^ (h << 16)
}

key         = 11011000000000000000000000000000
hash        = 01011000000000000000000000000000
slot        = 0 // 0101100000000000000000000 & 1111 (table size is 16 in this example)
fingerprint = 0

key         = 01010110000000000000000000000000
hash        = 00110110000000000000000000000000
slot        = 0 // 0011011000000000000000000 & 1111 (table size is 16 in this example)
fingerprint = 0
```

Note that the hash function has barely changed the key at all - the multiplication is not terribly effective as 
expected, but where other libraries use a xor-right-shift to introduce a little more entropy to lower bits, AndroidX's
xor-left-shift accomplishes nothing. The xor-left-shift is much more useful when more of the entropy is in the lower 
bits, as is the case for lowBits keys for example. The resulting slot and fingerprint are exactly the same, meaning 
we've achieved the absolute worst case - many keys all mapping to the same slot (and then distributed via geometric 
probing), and each fingerprint matching every probe (so every group of metadata has to be explicitly compared with 
the key which lives in a different array).

### HPPC highBits Performance

HPPC is in many ways identical to Fastutil - same basic structure, same hash finalizer, same algorithms. So why does 
its performance differ so drastically from Fastutil (and the other similar libraries) specifically for highBits 
keys? The answer lies in the details - we are looking at the geomean between both Int → Int and Long → Int maps. For
Int → Int maps HPPC does indeed use the same hash finalizer as Fastutil and other maps, and sees identical 
performance. However for Long keys, HPPC uses a different finalizer:

```java
var h = key * 0x9e3779b97f4a7c15L  // PHI
return (int) (h ^ (h >>> 32))
```

Compare this to the Long key finalizer used by Fastutil and other libraries:

```java
var h = key * 0x9e3779b97f4a7c15L  // PHI
h = h ^ (h >>> 32)
return (int) (h ^ (h >>> 16)) // second fold shifts entropy lower
```

### All Results

At this point, the only thing left is all the raw benchmarking results - feel free to continue scrolling if those 
are of interest, otherwise there's nothing more to read.

#### Int → Int / getHit

{{< benchmark-chart benchmark="IntMap.getHit" order="random" title="Int → Int / getHit — random keys" >}}
{{< benchmark-chart benchmark="IntMap.getHit" order="lowBits" title="Int → Int / getHit — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.getHit" order="highBits" title="Int → Int / getHit — highBits keys" >}}

#### Int → Int / getMiss

{{< benchmark-chart benchmark="IntMap.getMiss" order="random" title="Int → Int / getMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.getMiss" order="lowBits" title="Int → Int / getMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.getMiss" order="highBits" title="Int → Int / getMiss — highBits keys" >}}

#### Int → Int / putHit

{{< benchmark-chart benchmark="IntMap.putHit" order="random" title="Int → Int / putHit — random keys" >}}
{{< benchmark-chart benchmark="IntMap.putHit" order="lowBits" title="Int → Int / putHit — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.putHit" order="highBits" title="Int → Int / putHit — highBits keys" >}}

#### Int → Int / putMiss

{{< benchmark-chart benchmark="IntMap.putMiss" order="random" title="Int → Int / putMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.putMiss" order="lowBits" title="Int → Int / putMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.putMiss" order="highBits" title="Int → Int / putMiss — highBits keys" >}}

#### Int → Int / removeAndPutMiss

{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="random" title="Int → Int / removeAndPutMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="lowBits" title="Int → Int / removeAndPutMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="highBits" title="Int → Int / removeAndPutMiss — highBits keys" >}}

#### Int → Int / forEach

{{< benchmark-chart benchmark="IntMap.forEach" title="Int → Int / forEach" >}}

#### Int → Int / naiveCopy + preAllocatedCopy

{{< benchmark-chart benchmark="IntMap.naiveCopy" title="Int → Int / naiveCopy" >}}
{{< benchmark-chart benchmark="IntMap.preAllocatedCopy" title="Int → Int / preAllocatedCopy" >}}

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
