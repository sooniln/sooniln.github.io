---
title: "Comprehensive JVM Primitive Hashtable Benchmarks 2026"
date: 2026-06-18
draft: true
ShowToc: true
math: true
---

## Introduction

Hashtables are one of the building blocks of computer science, and deservedly get a lot of attention - but less so
within the JVM ecosystem. Part of this may simply be that most Java code is not written with performance top of mind,
or that the default JRE implementation is all-around quite good - reasonably efficient, resistant against attacks, 
etc... Today however, we'll be focusing entirely on performance.

For today, we're interested specifically in hashtables for primitive values. By focusing on values rather than
references we'll entirely ignore the cost of heap allocation, pointer dereferencing, GC pressure, and also come up 
with an analysis that will hopefully continue to hold weight with upcoming
[Valhalla](https://openjdk.org/projects/valhalla/) changes to the JVM.

Hashtables that store values rather than references can provide a range of large performance benefits within the JVM,
such as no boxing, no pointer indirection, better cache locality, etc... The JVM ecosystem has accumulated a handful of
libraries that address this by providing maps backed by primitive arrays. The problem is that the performance 
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
investigate in detail the design choices that affect hashtable performance.

Full benchmarking code used here is available at https://github.com/sooniln/jvm-collections-benchmarks.

## Hashtable Design Choices

Before we begin, it's worth starting with a quick refresher on how hash tables work, and the various design choices that
have sprung up over the years.

Pretty much all general purpose hashtables operate by mapping a very large universe of potential keys onto a much
smaller number of available slots (represented by in-memory data structure, almost always an array). As a trivial
example, if we're using integers as keys, we would need to map all the ~4 billion possible integers onto the much
smaller array in memory (for example a 128 slot array perhaps, if we're only expecting to map 100 integers total). This
accomplished via [hash functions](https://en.wikipedia.org/wiki/Hash_function).

Now by definition, once we've computed such a mapping there may be collisions - two different keys that are mapped 
by a hash function to the same slot. The second responsibility of the hash table is this to handle hash collisions, and
store multiple keys that map to the same slots somehow. This brings us to our first design decision,
[open addressing](https://en.wikipedia.org/wiki/Open_addressing) vs
[separate chaining](https://en.wikipedia.org/wiki/Hash_table#Separate_chaining).

### Open Addressing vs. Separate Chaining

The first architectural fork in hashtable design is how collisions are handled when two keys map to the same slot.

**Separate chaining** associates a separate collection of entries for each colliding key belonging to the same slot. 
Hash collisions are thus naturally handled by the collection. The advantage here is simplicity - it's trivial to add 
and remove things, and the selection of different collection types impacts performance. The most common collection 
used here is a linked list, but other options are possible (the JRE HashMap starts with a linked list for example, but
converts to a red-black tree if the list grows too large). The downside however is an additional indirection to 
enter the collection, and the cost of lookup within the collection (using a linked list for example requires pointer 
traversal and is very cache unfriendly).

**Open addressing** stores all entries directly in the contiguous backing array. When a slot is occupied, the algorithm
probes for the next candidate slot according to some probing strategy (discussed in more detail below). Lookups and 
insertions are often likely to stay nearby the original slot (cache friendly), which is why open addressing dominates 
value hashtables (which don't have the built-in pointer dereference cost of references). Deletion in an open 
addressed table however requires extra thought, and is discussed in more detail below.

Every library in this benchmark except the JRE HashMap uses open addressing (since every library except the JRE HashMap
is specialized for primitive keys).

### Load Factor

The [load factor](https://en.wikipedia.org/wiki/Hash_table#Load_factor) of a hashtable is the ratio of occupied 
slots to total capacity. At load factor 0.5, half the backing array is empty. At 0.9, only one slot in ten is free.
Most hashtables have a maximum load factor - when they reach that limit the backing array is expanded to reduce the
load factor.

Higher load factors conserve memory but increase expected probe sequence length, which grows non-linearly as the table
fills. For a uniformly random hash function and simple [linear probing](https://en.wikipedia.org/wiki/Linear_probing),
expected probe length at load factor α is approximately:

* Successful lookup (hit) ≈ $\frac{1}{2}\left(1 + \frac{1}{1-\alpha}\right)$
* Unsuccessful lookup (miss) ≈ $\frac{1}{2}\left(1 + \frac{1}{(1-\alpha)^2}\right)$

At α=0.5 this means ~1.5 probes for hits and ~2.5 for misses. At α=0.75 that's ~2.5 probes for hits and ~8.5 for misses.
And at α=0.9: ~5.5 for hits, ~50.5 for misses! This is a very simple calculation, assuming a perfect hash function and
the simplest possible version of linear probing, but it is useful for understanding general probe length behavior and
how important a good hash function is for linear probing.

### Backing Array Sizing

Open-addressed hashtables need to decide how large of an array to use to hold some number of entries at a given load 
factor. There are two main strategies.

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
with the prime table size (by definition), this results in a much more even spread of keys within the table with less
clustering. Less clustering reduces average probe length, which reduces lookup times. There is one additional advantage
to prime sizing - for non-linear probing schemes such as quadratic probing (discussed later) it is much easier to  
ensure no infinite loops in probing (every slot will be visited by the probe before repeating).

Trove is the only library in this benchmark using prime-sized tables. It also uses the identity hash for keys (no
mixing/avalanching at all), so prime sizing is doing some real work there to ensure a reasonable spread.

All other libraries use power-of-two sizing with a hash mixing step to spread key bits before the AND operation. Mixing
is discussed in greater detail further below.

### Array Memory Layout

Hashtables need to store both keys and value. There are two generally possible layouts:

**Parallel arrays** maintain one array for keys and a separate array for values:
keys:   \[k0]\[k1]\[k2]\[k3]...
values: \[v0]\[v1]\[v2]\[v3]...

**Interleaved storage** packs key-value pairs together:
table: \[k0]\[v0]\[k1]\[v1]\[k2]\[v2]\[k3]\[v3]...

Note that interleaved storage is only reasonable when keys and values are the same size - expanding a smaller type to a
larger one would waste space (and thus cache). It's not impossible strictly speaking, but generally just isn't worth 
it.

The advantage of separate arrays is that when a hashtable is probing through keys, more keys will fit in a single cache 
line, resulting in less cache misses. However once you've located the correct key, you still need to load the value -
with separate arrays the value is located somewhere else entirely, and you're incurring another possible cache miss to
load the value. Using interleaved storage means that once you locate the key, the value is immediately adjacent in
memory - no extra load necessary. But while probing, every cache line can now hold less keys, and thus more cache
misses are incurred...

As general rule we might hypothesize that separate arrays advantages lookups for keys that are not present, and
interleaved storage advantages lookups for keys that are present (or with short probes). Of course the only way to 
verify this is, you guessed it, benchmarking!

#### Metadata

There's one more wrinkle to add to the question of how to store data in memory - in addition to keys and values, some
hashtables also store additional metadata for each entry. For example,
[Robin Hood](https://en.wikipedia.org/wiki/Robin_Hood_hashing) hashtables may store probe sequence lengths (PSLs - how 
far a key is located from the slot it hashed too), and [Swiss tables](https://abseil.io/about/design/swisstables) store
key fingerprints. Given that that metadata is usually substantially smaller than the key size, metadata cannot be
interleaved with keys. If probing now requires accessing both metadata and keys this will incur many more cache misses,
so when metadata is used probing will almost always attempt to only access metadata, and access keys only when
absolutely necessary.

There is one more advantage to metadata - it can help with representing empty slots, which will be discussed in the
next section.



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

### Empty Slots

We've discussed key arrays, value arrays, interleaved key/value arrays, and separate metadata arrays. No matter which
is used however, a hashtable needs some way to indicate an empty slot, so that it know where it can insert new 
entries or when it can stop probing during a lookup. The problem of course, is that there is no guarantee any  
particular key value you might choose to represent "unoccupied" won't be inserted! There are naturally many possible 
solutions to this.

1. Enforce that the user provides an illegal key value which will always represent empty - the user can never insert
   that key into the map.
2. Use a separate variable to track whether the empty key value has been added to the map and what its value is. In
   every hashtable operation, check if the given key is the same as the empty key, and if so special-case the operation to
   use the separate variable instead of looking in the table.
3. Always allocate an array with 1 additional slot at the end, which is used to hold the empty key/value. Outside of
   this one slot, the empty key means empty for the rest of the array.
4. If the user attempts to insert the empty key, choose a new empty key at random which is not present in the table, and
   update the table to use the new empty key before continuing the insertion.
5. If using a metadata array, represent the empty slot in the metadata array. Now you don't need to care about the key
   array representing empty slots at all.

There are of course potential performance implications to all of these choices. If you need to add special cases to
public APIs you'll add to code size - too large of a code block can prevent some JVM C2 compiler optimizations and 
incur more code cache misses. If you need to check a member variable this is an additional load and register usage. Too
much register usage can cause register spilling and commensurate performance impacts on the hot path. If you increase
the array size by one, you may break C2 compiler knowledge about the range of your index variables, and prevent C2 from
eliding array range checks on access... No matter the choice there's always a variety of consequences to think through,
and pretty much no way to anticipate the impact except through a concept we haven't mentioned at all yet
(benchmarking).

### Probing Strategies

So far we've generally covered how data is represented internally, and how the map converts a hash into a table slot,
but what happens when that slot is already occupied (a hash collision)? In the case of separate chaining, we simply add
the new entry to the collection of entries already associated with the slot, but what about open addressing?

When a slot is occupied, the table must find another. The sequence of slots tried in the search for an open slot is
called the *probe sequence*, and its choice significantly impacts both throughput and cache behavior.

The simplest choice is **Linear probing**, which just tries consecutive slots: slot, slot+1, slot+2, and so on (wrapping
around the table) until an open slot is found. It is maximally cache-friendly (sequential memory access) — but suffers
from *primary clustering*: once a run of occupied slots forms, any key that hashes into
that run lengthens it, creating a feedback loop that lengthens the run further and degrades performance as entries are
put into slots further and further away from their home slot, and thus requiring longer and longer probe sequences on
lookup.

```
State: [][][A][B][C][][]

Insert D (hashes to slot 2): probe 2→3→4→5, insert at 5.
State: [][][A][B][C][D][]

Insert E (hashes to slot 5): probe 5→6, insert at 6. Cluster grew.
State: [][][A][B][C][D][E]
```

**[Quadratic probing](https://en.wikipedia.org/wiki/Quadratic_probing)** uses offsets slot+0², slot+1², slot+2², slot+3²... (the probe number squared). It avoids 
primary
clustering but can instead produce secondary clustering (keys with the same initial slot follow identical probe
sequences) and over long probe sequences results in more cache misses than linear probing.

**Double hashing** uses a second hash function to calculate the step size based on the key: slot, slot+step,
slot+2\*step, slot+3\*step, etc. It avoids both primary and secondary clustering - the cost is computing two hash
functions and the non-sequential memory access pattern.

**Robin Hood hashing** is a refinement of linear probing that swaps entry positions (swapping a "poorer" \[further from
it's home slot\] entry with a "richer" \[closer to it's home slot\] entry - hence Robin Hood...) during insertion to
reduce probe length variance. Entries that were "lucky" and landed near their ideal slot yield their position to
"unlucky" entries that have traveled further. This equalizes probe lengths and allows higher load factors without the
worst-case blow-up of plain linear probing.

And finally of course, you can have arbitrarily complex combinations of any probing scheme. A table could linear probe
for the first N elements, then switch to quadratic probing, or linear probing with a different hash function, etc... The
advantage to more complex probing schemes is that you can attempt to equalize the various pros and cons of each. For
example, by starting with linear probing you can take advantage of its cache friendly behavior initially - linear
probing for a cache line distance to avoid extra cache misses - then switch to quadratic probing when you're going to
incur a cache miss anyway.

| Library              | Probing Strategy                                                                   |
|----------------------|------------------------------------------------------------------------------------|
| JRE                  | Separate chaining (linked list -> red black tree)                                  |
| FastCollect          | Robin Hood + Combination (linear probing on hash1 -> linear probing on hash2)      |
| Fastutil             | Linear probing                                                                     |
| AndroidX             | Quadratic probing (each probe checks a group of 8 linear entries)                  |
| Trove                | Double hashing                                                                     |
| Koloboke             | Linear probing                                                                     |
| Eclipse              | Combination (linear probing on hash1 -> linear probing on hash2 -> double hashing) |
| HPPC                 | Linear probing                                                                     |
| Agrona               | Linear probing                                                                     |
| PrimitiveCollections | Linear probing                                                                     |

### Tombstones vs. Backward-Shift Deletion

Deleting a key from an open-addressed map is more complex than insertion. Simply
clearing the slot would leave a gap in a probe sequence, causing lookups for later
keys to terminate early at the gap and incorrectly report "key not found".

The standard solution is a **tombstone**: rather than clearing the slot, mark it with
a DELETED sentinel. Future lookups probe past tombstones. Future insertions can reuse
them. Most libraries in this benchmark use tombstones, including fastutil, Eclipse
Collections, HPPC, Agrona, and PrimitiveCollections.

The problem with tombstones is accumulation. A map with many insertions and deletions
will fill up with tombstones, which probe sequences must scan through even though they
represent no data. This degrades all operations over time. Most implementations
address this by triggering a rehash when tombstones exceed some fraction of capacity,
but this means delete-heavy workloads periodically pay a full-table rehash cost.
After deleting A and B:

[][][D][C][_][T][T]     (T = tombstone)

Lookup for E (hashes to slot 0): probe 0,1,2,3,4,5,6 — must scan tombstones

**Backward-shift deletion** (also called back-shift or Robin Hood deletion) avoids
tombstones entirely. After clearing a slot, the algorithm looks at the next slot and
checks if its occupant could shift back one position (i.e., its probe distance is
greater than zero — it's not at its ideal slot). If so, it shifts back. This
continues until an empty slot or a slot whose occupant is already at its ideal
position is reached.
Before: [][A(d=0)][B(d=1)][C(d=2)][]   (d = displacement from ideal)

Delete A:

[][][B(d=1)][C(d=2)][]

Shift B back (d=1, can move):

[][B(d=0)][][C(d=2)][]

Shift C back (d=2, can move):

[][B(d=0)][C(d=1)][][_]

The result is a table with no tombstones and no gaps — every probe sequence remains
contiguous. Lookup and insertion performance doesn't degrade after heavy deletion.
The cost is that deletion is slightly more expensive (the shift work), and backward-
shift deletion requires that entries know their displacement, which Robin Hood hashing
naturally maintains.

FastCollect uses backward-shift deletion, consistent with its Robin Hood probing
strategy. The Swiss Table design used by AndroidX uses a different approach: the
metadata byte for a deleted slot is set to a DELETED marker (0b11111110), and a
"drop deletes without full rehash" pass reclaims them by rehashing in place —
avoiding tombstone accumulation without requiring backward shifting.

### Hash Function Choice: Avalanche and Bias

For a primitive integer map, the key itself is an integer. A naive implementation
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
hashing](https://en.wikipedia.org/wiki/Hash_function#Fibonacci_hashing)** followed by an xorshift:

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

---




## Benchmark Infrastructure

* No CPU pinning, ran on idle machine.
* 6 Core AMD Ryzen 5 9600X - Windows 11

Benchmarks were run until standard deviation to mean ratios were at a reasonable level (where possible - there were a
few benchmarks that still did not converge nicely after a reasonable number of re-runs), then median was used to 
score in order to avoid outliers.

Two types of maps tested:

Integer → Integer
* Allows interleaved key/value layouts.
* Likely the most common type of primitive map.

Long → Integer
* Keys and values are of different size, interleaved layouts are unlikely to be useful.
* Key is of maximum primitive size (cannot be widened to accommodate metadata).
* Larger key size increases cache pressure.

For benchmarking, the load factor of every library was adjusted to a minimum of .75 for comparison (except for Eclipse,
which does not support this):

| Library              | Default | Benchmarked | Adjustable |
|----------------------|---------|-------------|------------|
| JRE                  | 0.75    | 0.75        | Yes        |
| FastCollect          | 0.9     | 0.9         | No         |
| Fastutil             | 0.75    | 0.75        | Yes        |
| AndroidX             | 0.875   | 0.875       | No         |
| Trove                | 0.5     | 0.75        | Yes        |
| Koloboke             | 0.667   | 0.75        | Yes        |
| Eclipse              | 0.50    | 0.50        | No         |
| HPPC                 | 0.75    | 0.75        | Yes        |
| Agrona               | n/a     | 0.75        | Yes        |
| PrimitiveCollections | 0.75    | 0.75        | Yes        |

Ten libraries were benchmarked. Versions are pinned as listed; the full benchmark code is available for independent reproduction.

## Libraries Under Test

### JRE

Java's standard library map uses separate chaining: each bucket holds a linked list that is automatically promoted to a
red-black tree once it grows too large, bounding worst-case lookup from O(n) to O(log n) per bucket under adversarial
key distributions. All keys are stored as boxed `Integer` or `Long` objects on the heap; every primitive key thus incurs
an allocation and a pointer indirection on access. Capacity is always a power of two; the default load factor is 0.75,
adjustable via the constructor.

The hash finalizer performs minimal mixing, relying on the red-black tree fallback behavior to prevent pathological
lookups.

```java
// int key
return key ^ (key >>> 16);

// long key
int h = (int)(key ^ (key >>> 32));
return h ^ (h >>> 16);
```

---

### FastCollect

*Disclosure: this post's author wrote FastCollect. See the introduction for full details.*

FastCollect is a Kotlin Multiplatform primitive collections library supporting JVM, JS, and native targets. Its maps use Robin Hood open addressing — an open addressing scheme that reduces lookup variance by keeping probe distances tightly balanced. On insertion, an incoming entry that has traveled farther from its ideal slot than the current occupant steals that slot. The `Int2IntHashMap` packs each key-value pair as a single `long` in one flat `LongArray` (key in the low 32 bits, value in the high 32 bits), halving the array count relative to separate key/value arrays and maximizing spatial locality. Tables use power-of-two capacity. The load factor is 0.9 for large tables and 1.0 for small tables up to two cache lines; it is not user-configurable. Tombstones are eliminated via backward-shifting on removal.

**Source:** [sooniln/fastcollect](https://github.com/sooniln/fastcollect)  
**Maven:** `io.github.sooniln:fastcollect-kotlin-jvm:2.0.0` *(development pre-release built from local source; latest published release: 1.0.1)*  
**Last release:** v1.0.1 — May 14, 2026

```kotlin
// int map: two-round XOR-shift/multiply
var h = key xor (key ushr 16)
h = h * 0x9E3779B9.toInt()
h = h xor (h ushr 16)

// long: XOR-fold to int, then same rotation scheme as int set
val h = (key xor (key ushr 32)).toInt()
// (key * 0x93d765dd).rotateLeft(log2Size) applied to h
```

---

### Fastutil

The most comprehensive and widely-adopted JVM primitive collections library. `Int2IntOpenHashMap` and
`Long2IntOpenHashMap` use open addressing with linear probing, power-of-two capacity, and a default load factor of 0.75,
adjustable via the constructor. Keys and values are stored in separate flat `int[]` / `long[]` arrays.

**Source:** [vigna/fastutil](https://github.com/vigna/fastutil)  
**Maven:** `it.unimi.dsi:fastutil:8.5.18`  
**Last release:** ~October 2025

The hash finalizer multiplies by the golden ratio constant PHI (from Knuth multiplicative hashing), but simply folds
high bits down into low bits for suboptimal mixing.

```java
// int key
static final int INT_PHI = 0x9E3779B9;
int h = x * INT_PHI;
return h ^ (h >>> 16);

// long key
static final long LONG_PHI = 0x9E3779B97F4A7C15L;
long h = x * LONG_PHI;
h ^= h >>> 32;
return h ^ (h >>> 16);
```

---

### Eclipse

Originally the Goldman Sachs Collections library, donated to the Eclipse Foundation in 2012. Eclipse maps use open
addressing with a three-phase probing strategy, power-of-two capacity, and an unchangeable load factor of .50. Key-value
pairs are stored interleaved if possible, and in separate arrays otherwise. For the three-phase probing strategy, phase
1 probes linearly for half a cache line based on hash 1; phase 2 probes linearly for half a cache line based on hash 2;
phase 3 falls back to double hashing.

Note: Eclipse forces a load factor of .50 - much lower than any other library in this benchmark and which cannot be
adjusted. Since this cannot be compared well to other libraries (which default to .75+), it is elided from graphs,
though it's raw benchmarking data is included (it is usually the fastest). Some experimentation with running other
libraries with a .50 load factor indicates they are competitive and may out-perform Eclipse, but true benchmarking has
not been performed.

**Source:** [eclipse/eclipse-collections](https://github.com/eclipse/eclipse-collections)  
**Website:** [eclipse.dev/collections](https://eclipse.dev/collections/)  
**Maven:** `org.eclipse.collections:eclipse-collections:13.0.0`  
**Last release:** December 24, 2024

The hash finalizer used here may be related to SipHash - not entirely clear where the constants come from.

```java
// int key — intSpreadOne (intSpreadTwo uses different constants)
code ^= code >>> 15;
code *= 0xACAB2A4D;
code ^= code >>> 15;
code *= 0x5CC7DF53;
code ^= code >>> 12;

// long key — longSpreadOne (longSpreadTwo uses different constants)
code ^= code >>> 28;
code *= -4254747342703917655L;  // 0xC4C882C7C77BA2C9L
code ^= code >>> 43;
code *= -908430792394475837L;   // 0xF36A08AC0D05E0C3L
code ^= code >>> 23;
```

---

### AndroidX

Google's Jetpack collection utilities library, now written in Kotlin and available as a Kotlin Multiplatform artifact.
`IntIntMap` and `LongIntMap` adopt a Swiss Table design. A separate array stores 1 byte of metadata per slot, organized
into groups of 8 (8 bytes = 1 long), and probes traverse this metadata array before entering separate key and value
arrays. The probe sequence is quadratic over groups, with power-of-two-aligned capacity; the load factor is locked at
.875.

For each byte of metadata, the low 7 bits store a per-slot fingerprint (H2) and the remaining bit encodes empty/deleted
state. Lookups consume metadata 8 entries at a time using SWAR (SIMD Within A Register) techniques, checking all eight
fingerprints in parallel before touching any key slot — dramatically reducing cache misses on a lookup miss. The top 25
bits of the hash (H1) drive the group probe offset; the low 7 bits (H2) serve as the fingerprint.

Note: AndroidX libraries are primarily intended for arm64, though they are distributed as multi-platform. Since
benchmarks were run on amd64, care should be taken not to generalize the results.

**Source:** [android.googlesource.com](https://android.googlesource.com/platform/frameworks/support) ([GitHub mirror](https://github.com/androidx/androidx))  
**Website:** [developer.android.com/jetpack/androidx/releases/collection](https://developer.android.com/jetpack/androidx/releases/collection)  
**Maven:** `androidx.collection:collection-jvm:1.6.0`  
**Last release:** March 11, 2026

```java
// int key: MurmurHash3 C1 multiply, spread low bits upward into high bits
val MurmurHashC1 = 0xcc9e2d51
int h = key * MurmurHashC1
return h ^ (h << 16)

// long key: XOR-fold to int, then same multiply-and-spread
int h = (key ^ (key >>> 32)).toInt() * MurmurHashC1
return h ^ (h << 16)
```

Let's quickly break down the Swiss table approach to metadata to see what this looks like. Swiss tables maintain a
separate **metadata array** — one byte per slot — that encodes slot state:

- 0b10000000: empty
- 0b11111110: deleted (tombstone)
- 0b0xxxxxxx: occupied, where the 7 bits `xxxxxxx` are the low 7 bits of the hash (the *fingerprint*)

Probing then works by scanning the metadata array rather than the key array and matching against the low 7 bits of the
hash. False positives on the fingerprint are possible, but unlikely, and it's only in the case of a fingerprint match
that it is necessary to check the key to fully confirm the match. In addition, because metadata bytes are small, 64
bytes (one cache line) covers 64 slots. With [SIMD](https://en.wikipedia.org/wiki/Single_instruction,_multiple_data) or SWAR (SIMD Within A Register) techniques, many metadata bytes can
be checked for a matching fingerprint simultaneously - much faster than iterating through them all one at a time. The
much cheaper cost of checking metadata also means that Swiss tables can tolerate much higher load factors without a
large performance impact.

---

### Trove

GNU Trove uses open addressing with double hashing: rather than a fixed linear step, each key computes a secondary
probe stride from its primary hash, so colliding keys scatter along independent paths and form shorter clusters than
linear probing would. Table capacity is always a prime number, ensuring the secondary stride is coprime to the table
size and that every slot is reachable. A separate byte array tracks each slot's state (FREE, FULL, or REMOVED). The
default load factor is 0.75, adjustable via the constructor.

**Source:** [bitbucket.org/trove4j/trove](https://bitbucket.org/trove4j/trove)
**Maven:** `net.sf.trove4j:core:3.1.0`  
**Last release:** August 3, 2018 (due to module split - no active development since ~2012 apparently)

The hash function for int keys is the identity — no bit mixing is applied; distribution quality relies entirely on the
double-hashing probe to counteract clustering. For long keys, the upper and lower 32-bit halves are XOR-folded to
produce a 32-bit hash.

```java
// int key: identity — no mixing
return value;

// long key: XOR-fold upper and lower halves
return (int)(value ^ (value >>> 32));
```

---

### Koloboke

Koloboke maps use open addressing with linear probing, power-of-two capacity, backwards-shift deletions, and a default load factor of 0.75 (user
configurable). Key-value pairs are stored interleaved if possible, and in separate arrays otherwise.

**Source:** [leventov/Koloboke](https://github.com/leventov/Koloboke) (archived)  
**Maven:** `com.koloboke:koloboke-impl-jdk8:1.0.0`  
**Last release:** May 26, 2016

The hash function is a Fibonacci-constant multiplicative mix
using the golden-ratio constants for 32-bit and 64-bit respectively, consistent with fastutil and HPPC.

```java
// int key
int h = key * 0x9E3779B9;
return h ^ (h >>> 16);

// long key (inferred from template structure — generated source not confirmed)
long h = key * 0x9E3779B97F4A7C15L;
return (int)(h ^ (h >>> 32));
```

---

### HPPC

High Performance Primitive Collections, maintained by Carrot Search. Maps use open addressing with linear probing,
power-of-two capacity, and a default load factor of 0.75, adjustable via the constructor. Keys and values are stored in separate flat arrays. Deletion
uses backward-shifting to avoid tombstones. Iteration uses a randomized starting offset (seeded per-instance) to prevent
consistent worst-case traversal patterns. The `mixPhi(long)` variant XOR-folds its result to `int` directly, unlike fastutil's `mix(long)` which preserves the full 64-bit result.

**Source:** [carrotsearch/hppc](https://github.com/carrotsearch/hppc)  
**Website:** [labs.carrotsearch.com/hppc.html](https://labs.carrotsearch.com/hppc.html)  
**Maven:** `com.carrotsearch:hppc:0.10.0`  
**Last release:** June 4, 2024

```java
// int key (BitMixer.mixPhi(int))
static final int PHI_C32 = 0x9e3779b9;
int h = k * PHI_C32;
return h ^ (h >>> 16);

// long key (BitMixer.mixPhi(long)) — XOR-folds result to int
static final long PHI_C64 = 0x9e3779b97f4a7c15L;
long h = k * PHI_C64;
return (int)(h ^ (h >>> 32));
```

---

### Agrona

Agrona is Real Logic's utility library, developed primarily to support the [Aeron](https://github.com/aeron-io/aeron)
messaging system. Primitive maps use open addressing with linear probing, power-of-two capacity, and a default load factor of 0.65, adjustable via the constructor (alongside the required missing-value sentinel). Key-value pairs are stored interleaved. Uses
backward-shift deletion on removal rather than tombstones. Every map must have an unrepresentable value passed in on
initialization (to mark empty slots), so Agrona can never represent maps with the full range of integers/longs. Note:
the long-key benchmark uses `Long2LongHashMap` (Agrona provides no `Long2IntHashMap`); values are widened to `long` at
the benchmark wrapper level.

Note: For benchmarking purposes, Agrona maps were initialized with a load factor of .75 rather than the default of .65.

**Source:** [aeron-io/agrona](https://github.com/aeron-io/agrona)  
**Maven:** `org.agrona:agrona:2.4.1`  
**Last release:** April 29, 2026

```java
// int key (Hashing.hash(int)) — two-round XOR-shift/multiply (Thomas Wang)
int x = value;
x = ((x >>> 16) ^ x) * 0x119de1f3;
x = ((x >>> 16) ^ x) * 0x119de1f3;
x = (x >>> 16) ^ x;
return x;

// long key (Hashing.hash(long)) — three-round with distinct constants (David Stafford)
long x = value;
x = (x ^ (x >>> 30)) * 0xbf58476d1ce4e5b9L;
x = (x ^ (x >>> 27)) * 0x94d049bb133111ebL;
x = x ^ (x >>> 31);
return (int)x ^ (int)(x >>> 32);
```

---

### Primitive Collections

Speiger's `Primitive-Collections` is a library explicitly aimed at duplicating and outperforming fastutil.
`Int2IntOpenHashMap` and `Long2IntOpenHashMap` use open addressing with linear probing, power-of-two capacity, and a
default load factor of 0.75, adjustable via the constructor. Keys and values are stored in separate flat arrays.

**Source:** [Speiger/Primitive-Collections](https://github.com/Speiger/Primitive-Collections)  
**Maven:** `io.github.speiger:Primitive-Collections:1.0.0`  
**Last release:** May 15, 2026

```java
// int key (HashUtil.mix(int))
static final int INT_PHI = 0x9E3779B9;
int h = x * INT_PHI;
return h ^ (h >>> 16);

// long key: XOR-fold via Long.hashCode(), then apply int mix
// HashUtil.mix(Long.hashCode(key))
int h = (int)(key ^ (key >>> 32)) * INT_PHI;
return h ^ (h >>> 16);
```
