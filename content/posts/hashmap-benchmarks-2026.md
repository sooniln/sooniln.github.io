---
title: "Comprehensive JVM Primitive Hashtable Benchmarks 2026"
date: 2026-07-18
draft: false
math: true
library:
  js:
    js1Hammer: https://cdn.jsdelivr.net/npm/hammerjs@2/hammer.min.js
    js2ChartJs: https://cdn.jsdelivr.net/npm/chart.js@4/dist/chart.umd.min.js
    js3ChartJsZoom: https://cdn.jsdelivr.net/npm/chartjs-plugin-zoom@2/dist/chartjs-plugin-zoom.min.js
---

## Introduction

Hashtables are one of the building blocks of computer science, and deservedly get a lot of attention - but less so
within the JVM ecosystem. Part of this may simply be that most JVM code is not written with performance top of mind,
or that the default JRE implementation is all-around reasonable - fast, efficient, resistant against attacks, why 
fix it if it ain't broke? Today we're looking at libraries that are interested in squeezing out every last byte and 
cycle of performance however, and we'll be focusing specifically on primitive hashtables.

If you need a hashtable with primitive keys and/or values, the default JRE HashMap will work fine, but since it 
deals exclusively with references your keys and value will be boxed. This means a large amount of memory overhead, 
and potentially additional CPU work (though as we'll see, modern JVMs have done a good job of driving this down).
Using primitive values rather than boxed references allows dropping the costs of heap allocation and pointer
dereferencing, as well as denser memory layouts and lower GC pressure. The hope is that any analysis here will 
continue to hold weight with upcoming [Valhalla](https://openjdk.org/projects/valhalla/) changes to the JVM, which 
will allow a much broader range of value objects. On a selfish note, it's also simpler to benchmark primitive 
hashtables as we don't have to worry about things like
[garbage collection rearranging arrays of references](https://shipilev.net/jvm/anatomy-quarks/11-moving-gc-locality/)
underneath us mid-benchmark and how that might affect results...

The JVM ecosystem has accumulated a handful of libraries that provide hashtables backed by primitive arrays in order
to address these performance concerns. The problem is that the performance tradeoffs and design choices between them 
are neither obvious nor well-documented anywhere. Many different choices are made around hash functions, collision 
resolution strategies, load factors, and those choices produce meaningfully different performance profiles yet there 
is pretty much no modern data available on head-to-head performance. The goal of this post is to rectify that.

We'll benchmark and analyze the following libraries:

* [JRE](#jre)
* [FastCollect](#fastcollect) (I am also the author of FastCollect)
* [Fastutil](#fastutil)
* [AndroidX](#androidx)
* [Trove](#trove)
* [Koloboke](#koloboke)
* [Eclipse](#eclipse)
* [HPPC](#hppc)
* [Agrona](#agrona)

Some of the questions we want to evaluate:

* How does load factor affect performance?
* How well does a JVM [Swiss Table](https://abseil.io/about/design/swisstables) (AndroidX) work?
* How does the novel (AFAIK) hash finalizer in FastCollect compare to more standard hash finalizers?
* How does the JRE HashMap compare to the more specialized libraries?

You may note that several of the libraries benchmarked here are quite old and/or no longer maintained. Others are 
not intended for general purpose usage. The larger purpose of this post is not to determine which is all-around 
fastest (and it's unlikely any one library could be characterized that way), but to investigate the design choices that 
affect hashtable performance.

> [!NOTE] Note
> The JRE HashMap is the only hashtable here truly suited for external use (when an attacker can control the table).
> Most of these libraries do not make any pretense at resisting DoS attacks and favor raw performance instead.

### Prior Benchmarks

The only large scale benchmarking of primitive hashtables I could find in the past is
[this article from 2014](https://dzone.com/articles/time-memory-tradeoff-example) by Roman Leventov (also the author of 
Koloboke). It's short, and worth a quick glance to observe how far behind the performance of JRE HashMap was in this
older benchmark - compare that to its performance today. Many of the libraries benchmarked in that article are also
benchmarked here (though some have changed names: HFTC → Koloboke, Goldman Sachs → Eclipse).

There is also the 2017 paper
[Empirical Study of Usage and Performance of Java Collections](https://research.spec.org/icpe_proceedings/2017/proceedings/p389.pdf),
but the paper itself generally only shows nothing more detailed than a faster/slower comparison, and I was unable to 
find the raw data referenced by the paper (I did not attempt to contact the authors in reference to the data).

## Hashtable Design Choices

Honestly, this section could have been an entire blog post in its own right. However, it doesn't feel right to drop 
into results without contextualizing how hashtables work and the various trade-offs we need to think about when 
designing them... If you feel you have a good handle on this and want to skip straight to the next section, move on 
to the [Benchmark Setup](#benchmark-setup) section. If not, I'll try to keep this short, but no guarantees.

Fundamentally, all hashtables operate by mapping a very large universe of potential keys onto a much smaller number of
available slots (often represented by an array). As a trivial example, if we're using 32-bit integers as keys, we would
need to map all the ~4 billion possible integers onto the much smaller structure in memory (for example a 128 slot 
array perhaps, if we're only expecting to map 100 integers total). This is accomplished via
[hash functions](https://en.wikipedia.org/wiki/Hash_function). By definition, once we've computed such a mapping there
may be collisions - two different keys that are mapped by a hash function to the same slot. The second responsibility
of the hashtable is thus to handle collisions, and store multiple keys that map to the same slots. These basic 
principles lead to certain tradeoffs.

### Tradeoffs

The highest level of tradeoff in hashtables can be characterized as time vs memory. Use more memory, and you can access
entries in less time. Use less memory, and it takes more time to access entries. Easy, but also a pretty simplified 
view of things. We can break this down further:

1. The CPU cost of mapping a key to a slot (complexity of the hash function)
2. The layout of slots in memory (open addressing vs separate chaining)
3. If the original slot mapping is a collision slot:
   1. The CPU cost of searching colliding slots (probing strategy)
   2. The number and pattern of memory accesses while searching colliding slots (probing strategy, cache line effects, 
      prefetching effects, TLB effects)
4. Second order effects
   1. Current load factor affects collision rates
   2. Key distributions affect collision rates
   3. Soft-deletion leaves behinds tombstones to reduce efficiency
   4. Size of keys/values determines how many fit in a cache line / cache
   5. Type of workload - reads vs writes, hits vs misses
   6. Etc, ad infinitum...

Each of these effects can vary on different invocations on the same hashtable, so the best to think about them is
in terms of the cost + variance they introduce. With that high-level framework in mind, let's dive into some quick-ish 
definitions.

### Open Addressing vs. Separate Chaining

The first architectural fork in hashtable design is how collisions are handled when two keys map to the same slot.

**Separate chaining** associates a separate collection of entries with each slot. All colliding entries thus are stored 
within the same collection. The most common collection used here is a linked list, but other options are possible 
(the JRE HashMap starts with a linked list for example, but converts to a red-black tree if the list grows too large).
This can be visualized as a two tier memory approach. 

**Open addressing** stores all entries directly in the contiguous backing array. When a slot is occupied, the algorithm
probes for the next candidate slot according to some probing strategy (discussed in more detail below). Deletion in 
an open-addressed table requires extra thought, and is discussed in more detail below. Where separate chaining can 
be thought of a two tier approach, open addressing is a flat one tier approach.

Every library in this benchmark except the JRE HashMap uses open addressing.

### Probing Strategies

When a key is mapped to a slot that is already occupied (a hash collision), it must be handled somehow. In the case of
separate chaining, we simply add the new entry to the collection of entries already associated with the slot, but 
open addressing must search for other unoccupied slots to store the entry. The sequence of slots tried in the 
search for an open slot is called the *probe sequence*, and its choice significantly impacts both throughput and 
cache behavior.

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
its home slot\] entry - hence Robin Hood) during insertion to reduce probe length variance. This tends to equalize
probe lengths and allows much higher load factors without the worst-case blow-up of plain linear probing (Robin Hood
hashtables are often run at load factors of 80%+ without much performance impact).

And finally of course, you can have arbitrarily complex combinations of any probing scheme. A table could use linear
probing for the first N elements, then switch to quadratic probing, or switch to linear probing with a different hash
function, etc... The advantage to more complex probing schemes is that you can attempt to equalize the various pros
and cons of each. For example, by starting with linear probing you can take advantage of its cache friendly behavior
initially - linear probing for a cache line distance to avoid extra cache misses - then switch to quadratic probing
when you're going to incur a cache miss anyway. The downside of these arbitrarily complex combinations is of course
code complexity - but also entry removal, which can become very difficult (and we'll discuss later).

### Load Factor

The [load factor](https://en.wikipedia.org/wiki/Hash_table#Load_factor) of a hashtable is the ratio of occupied 
slots to total capacity. At load factor 0.5, half the backing array is empty. At 0.9, only one slot in ten is free.
Most hashtables have a maximum load factor - when they reach that limit the backing array is expanded to reduce the
load factor again. This provides a convenient bound on performance, and allows the hashtable to scale memory usage
with the number of entries. It also means that performance varies as the hashtable grows from low load factors 
to high load factors, and then resizes back into a low load factor, leading to a distinct sawtooth graph appearance 
we'll observe later.

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

Array backed hashtables also need to decide how large of an array to use to hold some number of entries at a given 
load factor. There are two main strategies.

**Power-of-two sizing** uses table sizes that are always powers of two: 16, 32, 64, 128, and so on. This is primarily
for performance reasons:

1) Multiplying/dividing by two is a simple shift operation.
2) Converting a hash to a slot index can be done with a single AND rather than a more expensive modulo operation:

```java
int capacity = 256;      // 0b100000000
int mask = capacity - 1; // 0b011111111
int slot = hash & mask;  // slot is now in [0, 256)
```

**Prime sizing** keeps the table size as a prime number: 17, 37, 79, etc... Converting a hash to a slot now requires a
modulo, and is thus slower. The case for prime sizing is that because the hash cannot have any hidden common factors
with the prime table size (by definition), this results in a much more even spread of keys within the table and thus 
reduces clustering. Less clustering reduces average probe length, which reduces lookup times (theoretically).

The simple implementations of both of these schemes require a large re-allocation and complete rehash (re-insertion 
of all elements) whenever the hashtable runs out of room. All hashtables benchmarked here do complete rehashes on
running out of space (some allow for tombstone-removal-only rehashes in the event that they use tombstones - covered
later).

### Hashtable Memory Layout

At a minimum, hashtables need to store keys and values. They can also associate extra metadata with keys/values - 
often stored as a separate array if present. The overall goal for performance is to increase the memory density of 
data required for lookup operations as much as possible. The question this leads to in hashtable design is then, 
what data does your lookup require? Scan through keys, then load only the required value? Scan though metadata, 
loading keys only if necessary, and then only the required value? Or scan through keys and values together?

**Parallel arrays** maintain one array for keys and a separate array for values:

* keys:   \[k0]\[k1]\[k2]\[k3]...
* values: \[v0]\[v1]\[v2]\[v3]...

**Interleaved storage** packs key-value pairs together:

* table: \[k0]\[v0]\[k1]\[v1]\[k2]\[v2]\[k3]\[v3]...

> [!NOTE] Note
> If keys and values are not the same size then they will need to be padded to achieve interleaved storage. This
> wastes space, and is difficult to achieve with the JVM anyway, so interleaved storage is generally only a
> possibility when the keys and values are known to be the same size.

The density of data affects the number of cache misses during probe sequences, and thus directly affects performance.

### Metadata

Some example of metadata stored by hashtables includes:

* [Robin Hood](https://en.wikipedia.org/wiki/Robin_Hood_hashing) hashtables may store probe sequence lengths (PSLs - how
far a key is located from the slot it hashed to).
* [Swiss tables](https://abseil.io/about/design/swisstables) store key fingerprints.
 
Given that the metadata is usually substantially smaller than the key size, metadata cannot easily be interleaved with
keys and is usually a separate array. If probing now requires accessing both metadata and keys this will incur many 
more cache misses, so when metadata is used, probing will almost always attempt to only access metadata (which is 
what makes key fingerprints so useful), and access keys only when absolutely necessary.

There are also hashtables that store indirect indices which point into separate key/value storage. This allows 
key/value storage to be much denser than the main hashtable, potentially saving space, at the cost of an additional 
level of indirection. This might allow a hashtable to use a load factor of 50% or lower, knowing that it will only 
apply to a small metadata table, while keys/values are packed tightly. None of the hashtables being benchmarked use 
this technique.

Finally, there is one more advantage to metadata - it can help with representing empty slots.

### Empty Slots

We've discussed the various memory layouts and tradeoffs, but no matter which is used, a hashtable needs some way to
indicate an empty slot, so that it knows where it can insert new entries or when it can stop probing during a lookup.
The problem of course, is that there is no guarantee any particular key value you might choose to represent
"unoccupied" won't be inserted! There are naturally many possible solutions to this.

1. Enforce that the user provides an illegal key value which will always represent empty - the user can never insert
   that key into the map.
2. Use a separate variable to track whether the empty key value has been added to the map and what its value is. In
   every hashtable operation, check if the given key is the same as the empty key, and if so special-case the operation
   to use the separate variable instead of looking in the table.
3. Always allocate an array with 1 additional slot at the end, which is used to hold the value of the empty key. 
   In the rest of the array, the empty key means an empty slot.
4. If the user attempts to insert the empty key, choose a new empty key which is not present in the table, and update
   the table to use the new empty key before continuing the insertion.
5. If using a metadata array, represent the empty slot in the metadata array, not in the key array. 

There are potential performance implications to all of these choices. If you need to add special cases, you'll add 
to code size - too large of a code block can prevent some JVM C2 compiler optimizations and also incur more code cache
misses. If you need to check a member variable that's an additional load and register usage. Too much register usage
lead to register spilling and commensurate performance impacts on the hot path. No matter the choice there's always a
variety of consequences to think through, and pretty much no way to anticipate the impact except through benchmarking.

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
**Bias**: Do any particular outputs or output patterns appear more often than they should (favoring some slots over 
others and thus increasing collisions)?
**Entropy**: Another way of discussing avalanching and bias (avalanching means each output bit should have an 
entropy close to 1 bit conditionally on a single input bit flip, bias means for N bit output, ideal entropy is N 
bits) - essentially just reframing these in information-theoretic terms.

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

Variants on Knuth multiplicative hashing are used by the FastCollect, Fastutil, AndroidX, Koloboke, and HPPC libraries. 
Knuth multiplicative hashing by itself does not have great avalanching properties and tends to concentrate entropy 
in higher bits (rather, the entropy is in the fractional part of the result, which since we are using integer
multiplication and PHI values adjusted by 2^32, ends up being the high bits of the result), so common variants add a
xor-shift operation to (1) improve avalanching (2) improve entropy in the lower bits:

```java
final int INT_PHI = 0x9E3779B9 // 2^32 / PHI, where PHI is the golden ratio

int hash(int k) {
    int h = k * INT_PHI;
    return h ^ (h >>> 16); // fold high bits into low bits
}
```

Knuth multiplicative hashing is only one hashing strategy however, and there exist many other possibilities, all 
with their own lists of upsides and downsides.

#### Hash Functions vs Hashtable Needs

Hash functions are useful far beyond hashtables themselves - and so the goals for an ideal hash function may not 
mesh perfectly with the requirements of a hashtable. This exhibits itself in two common ways:

1) General purpose hash functions treat each bit as equally important - we want to spread entropy around everywhere. 
   But in reality, when we convert a hash to an actual table slot (which ranges from 0-N), we're usually throwing away
   everything except the lower bits. So in reality, entropy everywhere is not actually what hashtables care about, 
   entropy in the bits the hashtable uses is more important. This is the realization that spurred the different hash 
   function in the FastCollect library (more detail in the FastCollect section).
2) Methods of evaluating the strength of a hash function are generally predicated on the assumption that keys will 
   be drawn from random input. If there are stronger guarantees on the key space, then perhaps not all of these 
   properties are useful. A common example of this, what if we can assume integer keys can be mapped to [0-N)? This 
   is not an uncommon assumption in many hashtable domains, and in this case the identity hash function (i → i) is 
   actually a perfect hash function for a hashtable - even though the identity hash exhibits pretty much no 
   independence (linear key relationships are preserved) and awful avalanching (flipping 1 bit of input = 1 bit changed
   in output).

With that, our not-so-short recap of hashtable designs and tradeoffs is complete - let's move on to benchmarking!

## Benchmark Setup

* No CPU pinning, ran on idle machine.
* 6 Core AMD Ryzen 5 9600X - Windows 11
* L1/L2/L3 Cache Sizes: 48KB/core, 1MB/core, 32MB shared
* Temurin JDK 21.0.11

Benchmarking code itself can be found at 
[sooniln/jvm-collections-benchmarks](https://github.com/sooniln/jvm-collections-benchmarks).

Benchmarks were run until standard deviation to mean ratios were at a reasonable level (usually < 15%), then the median 
was used to score in order to avoid outliers.

Two types of maps tested:

Integer → Integer
* Same size for keys/values allows for interleaved memory layouts without penalty.
* Smaller keys pack more into cache.
* 32-bit keys are likely the most common in production.

Long → Integer
* Different size for keys/values prevents interleaved layouts.
* Larger keys increase cache pressure.
* 64-bit keys are likely the second most common in production (it seems unlikely that byte/short keyed hashtables 
  are very common).

The hashtables under test come with a variety of different default load factors. We face a choice of whether to try and
force similar load factors for a more apples-to-apples comparison, or use the default load factors which may 
unfairly advantage some implementations. Load factors have become an important part of hashtable design - it's a 
valid choice to use a low load factor for performance and recover memory in other ways (by ensuring the load factor
only affects the smaller metadata array and packing keys/value tightly outside the main array for example). However, 
none of the hashtables benchmarked here apply any such memory optimizations, and in the interest of similar 
comparisons we try to enforce that every hashtable has a load factor of at least 75% (higher is allowed). There is one
exception, Eclipse does not allow for load factor adjustments, and remains at 50% (and for this reason you'll note 
some asterixing of Eclipse results). We'll dig into the performance implications of this later.

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

### Key Selection and Ordering

Because hash functions have such a large effect on the performance of hash tables, it's important to test with data 
that can represent real world scenarios. We specify 3 key orders to test under:

* **random**: Keys are selected at random from the entire universe of possible keys. This might represent a scenario 
  where only a subset of keys are stored - for example if a hashtable is being used as a cache.
* **lowBits**: Keys are selected from 0 → table size and shuffled randomly. This has the effect that only the least 
  significant bits of the key tend to be set, and most significant bits are all zero. This might represent a scenario 
  where keys are generally linear and almost all keys are stored - for example internal id values.
* **highBits**: The same as lowBits, but every key is bit-reversed, so that the most significant bits tend to be set,
  and the least significant are all zero. This represents two scenarios - (1) an easy adversarial attack (since the 
  vast majority of hashtables are power-of-two sized, we know that they will usually truncate high bits in order to 
  calculate a slot - without good bit-mixing properties these tables may be vulnerable) (2) structures that have been 
  bit-packed into keys and thus have unusually structured bits (for example, a Point class might be trivially 
  encoded as a 64-bit value).

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
that were also not covered in any benchmarking - it's impossible to cover everything.

### Libraries Under Test

#### JRE

Java's standard library hashtable uses separate chaining: each bucket holds a linked list that is automatically
promoted to a red-black tree once it grows too large, bounding worst-case lookup from O(n) to O(log n) per bucket
under adversarial key distributions. All keys are stored as boxed primitive objects on the heap; every key thus incurs
an allocation and a pointer indirection on access.

* **Storage Schema**: Power-of-two sized, array of node pointers holding boxed keys/values (can form a linked list or
  red-black tree)
* **Probing Strategy**: Separate chaining (linked list/red-black tree)
* **Deletion Strategy**: Removal from linked list/red-black tree
* **Default Max Load Factor**: 75%

**Integer key hash finalizer**: The hash finalizer performs minimal mixing, relying on the red-black tree fallback
behavior to prevent pathological lookups.

```java
return key ^ (key >>> 16);
```

---

#### FastCollect

*Disclosure: I am the author of FastCollect.*

FastCollect is a Kotlin Multiplatform primitive collections library supporting JVM, JS, and native targets. The map
implementation currently uses Robin Hood hashtables because I wanted to play around with them.

**Source**: [sooniln/fastcollect](https://github.com/sooniln/fastcollect)  
**Maven**: `io.github.sooniln:fastcollect-kotlin-jvm:2.0.2`  
**Last release**: July 2026

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear probing (Robin Hood)
* **Removal Strategy**: Backwards-shift deletion
* **Default Max Load Factor**: 83.3% (100% for very small tables)

**Integer key hash finalizer**: FastCollect takes a different approach to multiplicative hashing than most of the
other libraries benchmarked. Several of the other libraries all use the same hash finalizer (which appears to have
originally come from the Koloboke library many years ago - `(key * PHI) ^ ((key * PHI) >>> 16)`). It seems worth
revisiting this finalizer as it exhibits several poor qualities that can be improved upon. While it is cheap, the
multiplication concentrates entropy in the high bits, yet it is only the low bits that will be used to derive the
slot. Some effort is made to ameliorate this with the xor-shift, but it seems insufficient. This finalizer exhibits 
worse results when given non-random inputs such as highBits.

For the FastCollect finalizer the key realization is that simply reversing the order of bits in the multiplication
result puts the most entropy exactly where we want it, in the low bits. The problem is that while bit-reversal is a
single instruction (RBIT) on ARM, x86-64 does not have a similar instruction. Divide and conquer + BSWAP approaches
and similar can make bit-reversal reasonably cheap, but not single cycle cheap like RBIT. Since benchmarking is
occurring on x86-64 we choose a slightly different approach: premix bits with xor-right-shift, multiply by PHI, and 
then reverse the bytes (not bits) with BSWAP/REV.

```java
var h = (key ^ (key >>> 16)) * 0x9E3779B9;  // PHI
return Integer.reverseBytes(h);
```

It would be very interesting to look into RBIT finalizers more on ARM platforms, it seems a much cheaper way of
achieving higher levels of entropy in a targeted part of the bit pattern, rather than focusing on achieving even
levels of entropy in all parts of the pattern (as more complex functions like murmur3 and so forth appear to do). As 
far as I can tell, using bit/byte reversal with Knuth multiplicative hashing appears to be novel, but it could be there 
are hidden weaknesses to this approach that I am not aware of.

---

#### Fastutil

One of the most comprehensive and widely-adopted JVM primitive collections libraries, Fastutil is very well known.

**Source**: [vigna/fastutil](https://github.com/vigna/fastutil)  
**Maven**: `it.unimi.dsi:fastutil:8.5.18`  
**Last release**: October 2025

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Max Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9;
return h ^ (h >>> 16);
```

---

#### AndroidX

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
* **Default Max Load Factor**: 87.5%

**Integer key hash finalizer**: Murmur3 finalizer. Note that the left shift instead of right shift would seem to
concentrate hash entropy further in the upper bits. This makes some sense for the Swiss Table design, where the
initial probe position is calculated from the high bits and the fingerprint is derived from the low bits. But this
opens up the risk of perhaps too little entropy in the low bits and thus the fingerprint (see benchmarking results for
highBits order)...

```java
var h = key * 0xCC9E2D51; // MurmurHashC1
return h ^ (h << 16);
```

---

#### Trove

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
* **Default Max Load Factor**: 50%

**Integer key hash finalizer**: The identity hash — no bit mixing is applied; distribution quality relies entirely on
the double-hashing probe and prime sized arrays to counteract clustering.

```java
// double hashing
// slot - identity hash
return key;

// stride
return 1 + (key % (length - 2));
```

---

#### Koloboke

A high performance collections library from Roman Leventov, appears to have been designed with HFT in mind. It makes
use of deprecated Unsafe APIs for performance.

**Source**: [leventov/Koloboke](https://github.com/leventov/Koloboke)  
**Maven**: `com.koloboke:koloboke-impl-jdk8:1.0.0`  
**Last release**: May 2016

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Max Load Factor**: 66.7%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9; // PHI
return h ^ (h >>> 16);
```

---

#### Eclipse

Originally the Goldman Sachs Collections library, donated to the Eclipse Foundation in 2012. Eclipse forces a load
factor of 50% - much lower than any other library in this benchmark - and it cannot be adjusted. Beware assuming
an apples to apples comparison. Some experimentation with running other libraries with a 50% load factor indicates
they are competitive and outperform Eclipse, but full benchmarking has not been performed.

**Source**: [eclipse/eclipse-collections](https://github.com/eclipse/eclipse-collections)  
**Website**: [eclipse.dev/collections](https://eclipse.dev/collections/)  
**Maven**: `org.eclipse.collections:eclipse-collections:13.0.0`  
**Last release**: December 24, 2024

* **Storage Schema**: Power-of-two sized, interleaved array if key/value size matches, parallel arrays otherwise
* **Probing Strategy**: Combination (linear probing on hash1 for half a cache line → linear probing on hash2 for
  half a cache line → double hashing)
* **Removal Strategy**: Tombstones - tombstones count towards rehashing load factor
* **Default Max Load Factor**: 50%

**Integer key hash finalizer**: 3 separate hash finalizers for the various probing steps. It's not entirely clear where
the constants come from.

```java
// hash1 - identity hash
return key;

// hash2 - murmur3 style finalizer
var h = key ^ (key >>> 14);
h *= 0xBA1CCD33;
h ^= h >>> 13;
h *= 0x9B6296CB;
return h ^ (h >>> 12);

// --double hashing--
// double hashing slot
var h = key ^ (key >>> 15);
h *= 0xACAB2A4D;
h ^= h >>> 15;
h *= 0x5CC7DF53;
return h ^ (h >>> 12);
// double hashing stride - odd stride ensures every slot is hit with power-of-two sized array
return Integer.reverse(hash2(key)) | 1;
```

---

#### HPPC

High Performance Primitive Collections, maintained by Carrot Search.

**Source**: [carrotsearch/hppc](https://github.com/carrotsearch/hppc)  
**Website**: [labs.carrotsearch.com/hppc.html](https://labs.carrotsearch.com/hppc.html)  
**Maven**: `com.carrotsearch:hppc:0.10.0`  
**Last release:** June 2024

* **Storage Schema**: Power-of-two sized, parallel arrays for keys/values
* **Probing Strategy**: Linear probing
* **Removal Strategy**: Backwards-shift deletion
* **Default Max Load Factor**: 75%

**Integer key hash finalizer**: Knuth multiplicative hashing with constant shift.

```java
var h = key * 0x9E3779B9; // PHI
return h ^ (h >>> 16);
```

---

#### Agrona

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
* **Default Max Load Factor**: 65%

**Integer key hash finalizer**: Two-round XOR-shift/multiply from Chris Wellons' Hash Prospector.

```java
var h = key ^ (key >>> 16);
h = h * 0x119DE1F3;
h = h ^ (h >>> 16);
h = h * 0x119DE1F3;
return h ^ (h >>> 16);
```

---

## Benchmarks

Raw benchmark data can be downloaded [here](/libraries.csv).

While viewing graphs, you may use the button to switch the time axis between log scale and linear scale. Clicking on 
a library in the legend will hide/show the data for that library. The mouse can be used to pan/zoom the graphs. 
Double-clicking a graph will reset its pan/zoom. Always keep in mind that the graphs default to a logarithmic scale 
which makes a relative comparison harder to eyeball. Hover over data points for actual numbers. You'll have a better
time looking at these graphs on a larger display (not your phone).

### Overall Results

Rather than forcing you to scroll past the excessive graphs and numbers below which go into individual scenarios, 
might as well cut to the chase, right?

#### Memory Usage

Memory usage is measured with the [JOL (Java Object Layout)](https://github.com/openjdk/jol) library. Note that 
several libraries use pretty much the exact same amount of memory, and thus their data series are overlaid on top of 
one another in the graph. Recall that clicking a library in the legend will hide/show its data series.

{{< benchmark-chart benchmark="IntMap.memory" title="Int → Int Map Memory Usage" >}}

Results are pretty much as we'd expect - JRE maps which store objects take up 3-6.5x as much memory as maps which store
primitives. The effects of prime sizing (Trove) vs power-of-two sizing (everyone else) and different load factors
are also on display.

{{< benchmark-chart benchmark="LongMap.memory" title="Long → Int Map Memory Usage" >}}

For Long maps results are relatively the same, except that Agrona now uses more memory (recall that as a 
special-purpose library Agrona does not support Long → Int maps, and thus we use its Long → Long map instead).

#### Read Performance

Read performance is calculated as the geometric mean of GetHit and GetMiss benchmarks. This weights both hits and 
misses equally, an assumption which may not be true in various real world workloads. Results for the three orderings 
(random, lowBits, highBits) follow:

{{< benchmark-chart benchmark="Map.readGM" order="random" title="Map Read Geometric Mean — random keys" >}}

Geomeans across all sizes (lower is better):

|                  | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |  HPPC |    JRE | Koloboke |  Trove |
|------------------|-------:|---------:|--------:|------------:|---------:|------:|-------:|---------:|-------:|
| Geomean (random) | 12.537 |    8.681 |  5.421* |       8.442 |   10.475 | 9.274 | 11.858 |   11.060 | 17.717 |

As this is the first graph we're looking at, note the distinct regime changes as the hashtable size increases past
L1/L2/L3 cache sizes. For many of the power-of-two sized graphs we see the distinct sawtooth performance that comes
from different loads as the table is queried when more empty vs more full (the sizes benchmarked were explicitly
chosen to differentiate this).

Looking at AndroidX, we can note a couple of things - (1) impressive read performance from ~32K entries onwards -
comparable with Eclipse operating at a much lower load factor (2) a more subdued change in performance as size
increases (thanks to a much smaller metadata array), and (3) it hits a performance cliff and times out around 32M 
entries. Further analysis of AndroidX can be found later.

We also note that JRE read performance is relatively competitive (although it uses far more memory of course). 
This is a drastic improvement from the benchmarking performed in 2014.

Eclipse is the obvious fastest across all sizes - however how much of its performance is due to the fact that it is 
using an unfairly lower (for comparison purposes) load factor? We'll dig into that later as well.

{{< benchmark-chart benchmark="Map.readGM" order="lowBits" title="Map Read Geometric Mean — lowBits keys" >}}

Geomeans across all sizes (lower is better):

|                   | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |  HPPC |   JRE | Koloboke | Trove |
|-------------------|-------:|---------:|--------:|------------:|---------:|------:|------:|---------:|------:|
| Geomean (lowBits) | 12.554 |    7.734 |  2.969* |       6.521 |    9.390 | 8.877 | 7.100 |    9.454 | 5.313 |

Recall that lowBits keys concentrate their entropy in the lower bits, which is where most of these tables calculate 
slot positions from. The biggest change we see here is with Trove performance (it was one of the slowest for random 
keys, but is now among the fastest for lowBits keys). As covered earlier, Trove uses the identity hash (which is 
"perfect" for lowBits keys), so it is unsurprising that it performs so well with them. Eclipse also improves its 
performance - Eclipse's first choice of hash finalizer is also the identity hash.

{{< benchmark-chart benchmark="Map.readGM" order="highBits" title="Map Read Geometric Mean — highBits keys" >}}

Geomeans across all sizes (lower is better):

|                    | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |   HPPC |    JRE | Koloboke |  Trove |
|--------------------|-------:|---------:|--------:|------------:|---------:|-------:|-------:|---------:|-------:|
| Geomean (highBits) | 12.619 |   71.019 |  8.944* |       6.565 |   18.002 | 61.927 | 17.151 |   18.079 | 18.083 |

Recall that highBits keys concentrate their entropy in the higher bits, which most of these tables ignore in order 
to calculate slot positions. So we expect this to be an interesting "adversarial" order, and the results bear that 
out.

AndroidX and HPPC in particular show pathological performance degradation, especially at smaller sizes, with 
AndroidX managing to take ~1600ns on average for a lookup in the absolute worst case. Fastutil also sees performance 
taking a hit, though not to the same extreme factor.

Interestingly, if we look at JRE performance, not only is it quite reasonable at lower sizes (though it's also 
brought to its knees at higher sizes), it's possible that we're able to see the effect of JRE's red-black tree fallback
ameliorating the performance hit at larger sizes (though I have not confirmed that's what's actually happening here)?

FastCollect performs sufficiently well with highBits keys that it is able to out-perform even Eclipse, though 
Eclipse is operating at a load factor of 50% and FastCollect at 83%.

#### Write Performance

Write performance is calculated as the geometric mean of the PutHit, PutMiss, and RemoveAndPutMiss benchmarks. This 
weights miss benchmarks higher, as they are represented twice (PutMiss and RemoveAndPutMiss), an assumption which may 
not be true in various real world workloads. Results for the three orderings (random, lowBits, highBits) follow:

{{< benchmark-chart benchmark="Map.writeGM" order="random" title="Map Write Geometric Mean — random keys" >}}

Geomeans across all sizes (lower is better):

|                  | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |   HPPC |    JRE | Koloboke |  Trove |
|------------------|-------:|---------:|--------:|------------:|---------:|-------:|-------:|---------:|-------:|
| Geomean (random) | 18.426 |   19.661 | 12.412* |      16.997 |   18.612 | 13.792 | 31.936 |   16.535 | 31.215 |

We immediately see some patterns which look the same as from the read results, and note that AndroidX again hits a 
performance cliff and times out at higher sizes. JRE is less competitive for writes than reads it would also appear,
and while Eclipse is again generally the fastest, it isn't as dominant as it was for reads (again with the caveat
that Eclipse is operating at a different load factor than all the other libraries).

{{< benchmark-chart benchmark="Map.writeGM" order="lowBits" title="Map Write Geometric Mean — lowBits keys" >}}

Geomeans across all sizes (lower is better):

|                   | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |   HPPC |    JRE | Koloboke |  Trove |
|-------------------|-------:|---------:|--------:|------------:|---------:|-------:|-------:|---------:|-------:|
| Geomean (lowBits) | 18.397 |   19.466 |  7.972* |      12.913 |   16.549 | 13.206 | 20.959 |   15.448 | 15.668 |

Again, similar patterns for lowBits in writes from reads. Trove performs better with lowBits keys than random keys 
(thanks to its identity finalizer), and the same for Eclipse (with its first choice identity finalizer).

{{< benchmark-chart benchmark="Map.writeGM" order="highBits" title="Map Write Geometric Mean — highBits keys" >}}

Geomeans across all sizes (lower is better):

|                    | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |    HPPC |    JRE | Koloboke |  Trove |
|--------------------|-------:|---------:|--------:|------------:|---------:|--------:|-------:|---------:|-------:|
| Geomean (highBits) | 18.470 |  220.477 | 18.939* |      14.154 |   34.261 | 139.099 | 70.280 |   27.543 | 36.299 |

And for highBits keys we see the same patterns for AndroidX and HPPC, awful performance, especially at lower sizes. 
Fastutil shows the same degradation, just not as bad.

#### Iteration Performance

Since iteration is agnostic to actual key values, it is simply the geometric mean of Int and Long keyed map iterations.

{{< benchmark-chart benchmark="Map.forEachGM" title="Int → Int / forEach" >}}

Geomeans across all sizes (lower is better):

|   Agrona | AndroidX |  Eclipse | FastCollect | Fastutil |     HPPC |      JRE | Koloboke |   Trove |
|---------:|---------:|---------:|------------:|---------:|---------:|---------:|---------:|--------:|
| 1180.189 |  394.426 | 1493.984 |    1136.482 |  996.386 | 1910.326 | 1566.672 |  917.517 | 826.078 |

From roughly ~16K entries onwards, JRE iteration is slower than every other library - a direct consequence of it 
using more memory. Accessing more memory in a linear fashion takes more time - no two ways about it. Without 
investigation, it's difficult to say why JRE iteration is competitive for fewer entries, but I would tend to assume 
cache effects?

On the opposite side, AndroidX has by far the fastest iteration up to ~16K entries - this is almost certainly due to 
the fact that skipping empty slots only requires touching the much smaller metadata array for AndroidX. Since it's 
so much smaller, it's much easier for it to stay in cache, even at larger sizes. Once a non-empty slot is 
reached however, AndroidX will still need to load from the key/value arrays. I would suspect that up to ~16K 
elements, metadata/keys/values are all able to stay in cache relatively well, but after keys/values drop out of 
cache, iteration slows.

HPPC however has unexpectedly slow iteration! This is a consequence of a design decision HPPC has made to prevent the
user from shooting themselves in the foot with naive copy operations. Most of these libraries under test iterate in the 
same order as elements appear in the key or metadata array via their hashcode, which means iteration indirectly exposes 
hash order. If iteration order is then used for insertion again (for example copying every element into 
another map), this can be a worst case scenario and 
[lead to quadratic runtimes](https://accidentallyquadratic.tumblr.com/post/153545455987/rust-hash-iteration-reinsertion)
(but the amelioration is also generally trivial, simply pre-size the map appropriately). In order to prevent 
potentially quadratic runtimes from occurring, HPPC uses a random step length for iteration - iteration no longer 
exposes hash ordering, but is also now not very cache friendly and thus is slower. Many of the other libraries 
benchmarked here suffer the same problem, but generally ignore it and allow the user to shoot themselves in the foot 
if they aren't careful with proper pre-sizing so that iteration is not artificially slowed.

Indeed, FastCollect suffers the same problem as well (to an even greater degree, given that maintaining the Robin 
Hood invariant requires extra work during insertion), but takes a slightly different approach. Any linear probing 
approach means we can expect calculable upper bounds on maximum run length for various map sizes (while there are 
indeed formulas to calculate this, see https://www.cs.tau.ac.il//~zwick/Adv-Alg-2015/Linear-Probing.pdf for more 
information, I skipped that and simply used Monte Carlo simulations). I.e. given a map with a backing array of 
length 65536 which is ~83.3% full, we would expect VERY roughly a maximum run length (consecutive non-empty slots) 
of ~500 at the 99.9th percentile. So if we see a run length > 500 in a map of that size, it's extraordinarily likely
that something is going drastically wrong. And further, if we're in the accidentally quadratic re-insertion 
scenario discussed above, we'll likely encounter that warning after only ~500 elements - quite early in the 
re-insertion! When FastCollect detects these pathologically long run lengths that should never be expected under any 
normal scenario, it will pre-emptively resize the map larger, with the effect of ameliorating quadratic runtimes. This 
adds slightly more complex logic to detect run lengths during insertion, but correct branch prediction minimizes the
cost of this and still allows a fast iteration implementation.

#### Copy Performance

Here we'll only examine pre-allocated copy performance, as we really discussed all the interesting bits of naive 
copy performance just now under the iteration section (though feel free to check out naive copy results as well 
under the [All Results](#all-results) section).

{{< benchmark-chart benchmark="IntMap.preAllocatedCopy" title="Int → Int / preAllocatedCopy" >}}

Geomeans across all sizes (lower is better):

|  Agrona | AndroidX | Eclipse | FastCollect | Fastutil |    HPPC |     JRE | Koloboke |   Trove |
|--------:|---------:|--------:|------------:|---------:|--------:|--------:|---------:|--------:|
| 774.117 |  576.580 | 490.935 |      18.716 |  138.692 | 620.683 | 259.533 |  121.097 | 288.608 |

There's perhaps not too much surprising here - copy performance is tied to write performance, except that
FastCollect is able to substantially outperform all other maps. FastCollect attempts to use direct memcpy whenever
possible in copying, and for the simple copy performed here (start with an empty map, copy all elements from another
map, which is already sized correctly) this yields much faster performance than manual iteration and insertion.

### Further Investigations

#### Load Factor & Eclipse Performance

Eclipse generally out-performs all the other libraries we've benchmarked - but we've hypothesized that this is due
to the fact that it uses a load factor of 50%, where every other library uses a load factor of 75% or higher. Is
this actually true though? Or is the load factor not actually the defining factor in Eclipse's performance? We couldn't
perform full benchmarking (limited time), but here are the results for read geomean performance with all libraries
set to 50% load factor (AndroidX excluded since load factor is not adjustable, and FastCollect using a build with 
load factor adjusted manually).

{{< benchmark-chart benchmark="MapLF50.readGM" order="random" title="50% LF Map Read Geometric Mean — random keys" >}}

Geomeans across all sizes (lower is better):

|                  | Agrona | AndroidX | Eclipse | FastCollect | Fastutil |  HPPC |   JRE | Koloboke |  Trove |
|------------------|-------:|---------:|--------:|------------:|---------:|------:|------:|---------:|-------:|
| Geomean (random) |  7.360 |      n/a |   5.900 |       5.469 |    5.869 | 5.778 | 9.344 |    5.669 | 12.022 |

With all libraries at 50% load factor, Eclipse is now solidly middle of the pack for read performance, and 
FastCollect is now the fastest by a small margin. We can conclude that load factor was indeed the driving reason behind 
Eclipse's performance.

At lower load factors like 50%, there's a higher probability that any given entry will be residing in its home slot 
(the slot it hashed to), meaning that this rewards libraries that have a fast home slot pathway. For a Robin Hood 
hashtable, this also strongly increases the probability that if any entry is not at its home slot, it can be found 
on the same cache line as the home slot, and thus reduces the probability of a cache miss.

#### AndroidX Oddities

AndroidX benchmarks:
1. Appear to hit a performance cliff at > ~32 million entries
2. Timeout at very high numbers of entries, so there is no data for > ~32 million entries
3. Exhibit extremely poor performance with highBits keys.

##### Performance Cliff At ~32 Million

With about 32 million entries, AndroidX's metadata array (1 byte per entry) jumps from ~32MB to ~64MB, and thus no
longer fits in the L3 cache of the benchmarking machine (32MB). Almost every probe is hitting cold memory
(perhaps further exacerbated by the geometric probing sequence?). It's worth remembering that AndroidX is primarily
intended for use on Android devices, which are likely to have a much smaller L3 cache than the desktop device used
for this benchmarking, but also are unlikely to be dealing with such large maps.

##### Benchmark Timeout

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

##### Performance Drop For HighBits Keys

In order to understand why AndroidX is so much more affected by highBits keys we'll need to dig into how exactly the 
library handles storage in the first place. AndroidX primarily uses a metadata array for lookups, with an equivalent 
key and value array. We'll focus only on the metadata array for now. When a key is hashed, the resulting 32-bit hash 
is separated into two parts:

* H1: the hash's 25 most significant bits - used to determine the initial slot via masking by table size
* H2: the hash's 7 least significant bits - stored as a fingerprint within the metadata array

Let's examine what happens with a highBits example:

```
int hash(int key) {
  var h = key * 0xCC9E2D51; // MurmurHashC1
  return h ^ (h << 16);
}

key         = 11011000000000000000000000000000
hash        = 01011000000000000000000000000000
slot        = 0 // 0101100000000000000000000 & 1111 (table size is 16 in this example)
fingerprint = 0000000

key         = 01010110000000000000000000000000
hash        = 00110110000000000000000000000000
slot        = 0 // 0011011000000000000000000 & 1111 (table size is 16 in this example)
fingerprint = 0000000
```

Note that the hash function has barely changed the key at all - the multiplication is not terribly effective as 
expected, but where other libraries use a xor-right-shift to introduce a little more entropy to lower bits, AndroidX's
xor-left-shift accomplishes nothing. The xor-left-shift is much more useful when more of the entropy is in the lower 
bits, as is the case for lowBits keys for example. The resulting slot and fingerprint are exactly the same, meaning 
we've achieved the absolute worst case - many keys all mapping to the same slot (and then distributed via geometric 
probing), and each fingerprint matching every probe (so every group of metadata has to be explicitly compared with 
the key which lives in a different array).

#### HPPC HighBits Performance

HPPC is in many ways identical to Fastutil - same basic structure, same hash finalizer, same algorithms. So why does 
its performance differ so drastically from Fastutil (and the other similar libraries) specifically for highBits 
keys? The answer lies in the details - we are looking at the geomean between both Int → Int and Long → Int maps. For
Int → Int maps HPPC does indeed use the same hash finalizer as Fastutil and other maps, and sees identical 
performance. However for Long keys, HPPC uses a different finalizer:

```java
var h = key * 0x9e3779b97f4a7c15L;  // PHI
return (int) (h ^ (h >>> 32));
```

Compare this to the Long key finalizer used by Fastutil and other libraries:

```java
var h = key * 0x9e3779b97f4a7c15L;  // PHI
h = h ^ (h >>> 32);
return (int) (h ^ (h >>> 16));  // second fold shifts entropy lower
```

HPPC's lack of an additional fold to spread entropy downwards leads to substantially worse performance for Long maps 
on highBits keys, and this is visible in the final geometric mean results.

### Final Takeaways

We've confirmed the basics - compared to the JRE HashMap, hashtables specialized for primitives may deliver up to:

1. **3-6.5x memory savings**
2. **2-3x CPU savings**

This isn't always guaranteed however - there are scenarios where the JRE HashMap is competitive with the more 
specialized hashtables for CPU usage. In the several different libraries we've explored, what stands out?

#### JRE

JRE HashMap is generally extremely competitive with primitive hashtables when it comes to GetMiss scenarios 
(performing better than many of the libraries benchmarked), but is far slower than all other libraries in GetHit 
scenarios. I'd hypothesis this is due to the two tier memory structure employed by separate chaining tables - 
they are often able to skip entering the second tier of memory (the linked list) entirely in GetMiss scenarios. 
If it's necessary to enter the second tier (GetHit and Put scenarios) however, JRE HashMaps cannot compete on speed 
(far more pointer dereferencing required, and likely non-linear memory access patterns depending on how the 
garbage collector has arranged memory).

In the 2014 benchmarks, HashMap was ~9x (very roughly eyeballed, don't take this too seriously) slower than 
Koloboke in GetHit scenarios. In today's benchmark, HashMap is only roughly ~2-3x slower! An impressive 
improvement over the years. We might expect efforts like
[Compressed Object Headers](https://openjdk.org/projects/lilliput/) (available since Java 24-25, but not in 21 
on which benchmarking was performed) to further reduce this distance.

---

#### FastCollect

I originally chose to implement Robin Hood hashing for the fun of it - there's a certain elegance about it, and I was 
unfamiliar with it. While it's worked out quite well, I am unsure that Robin Hood hashing is worth it in the 
long run. There are certainly scenarios it shines in, but I think for maximum performance simple linear probing 
without any hoopla may get closer to allowing the hardware to operate the way it wants to with maximum performance.
Linear scanning through memory is hard to beat.

Simple linear probing without Robin Hood actually has more entries sitting at their home slots than a Robin Hood
table (since entries sitting at their home slots are considered "richest", and thus "stolen from" more often). This
means that in GetHit scenarios, a non-RH table has (1) more entries that require only a single memory load since
they are sitting at their home slot (2) entries that are not sitting at their home slot tend to require more
probes to reach. However, since probe length increases logarithmically with table size (and since prefetching
can make linear accesses cheaper), this may not be as large a disadvantage as it first appears... My overall
takeaway is that Robin Hood tables may in the long run not be worth the trouble, but I don't have perfect clarity on
this. It's very unclear how much of a performance difference is coming from the RH invariant vs the different
hash finalizer being used, and more experimentation is necessary.

The idea of using bit/byte reversal in the hash finalizer appears to have been validated, as it out-performed more 
standard finalizers for FastCollect specifically. As far as I can determine, this appears to be novel (I couldn't
find any information on any other implementations using this technique), yet it performed very well. It's still
possible someone with a deeper background in hashing than me could point out some fatal flaw. I am curious about
bringing this family of bit-reversing hash finalizers to a simple linear probing implementation - would that be 
even faster?

The additional variables / logic needed by Robin Hood hashing make it much harder to avoid register spilling in
inlined methods on the hot path. It took quite some effort to arrange code to try and encourage C2 compilation
patterns that avoided unnecessary register spilling and the commensurate performance impact. Simpler code that
doesn't enforce Robin Hood invariants tends to reduce the likelihood of this scenario occurring.

Overall FastCollect's performance was among the fastest benchmarked, but there are avenues for further exploration.

---

#### AndroidX

The AndroidX Swiss table design is clearly excellent for GetMiss scenarios at larger sizes, but average or poor at 
most others. GetMiss demonstrates AndroidX's strengths:

* AndroidX is very fast at iterating through groups of 8 entries at a time, and it can check if those 8 entries 
  may contain the lookup key in effectively a single operation. If a group of 8 entries may contain the key 
  however, AndroidX has to use more expensive bit-twiddling techniques to confirm the match and then extract 
  the correct entry.
* Even so, GetMiss performance is only advantageous outside the L2 cache regime (L3+). If the hashtable fits in L2 
  cache, AndroidX tends to be slower than other tables for GetMiss. I would hypothesize that below this size, 
  a combination of shorter run lengths (fewer entries to iterate through) and cheap cache misses (loading the
  next entry hits L1/L2) means that AndroidX's geometric probing is at a disadvantage. Other libraries which use 
  linear probing may also be benefiting more from prefetching?

C++ is largely dominated by Swiss tables (and similar designs that use SIMD APIs) these days - Java isn't. 
Unfortunately today, 8 years after Swiss tables were first released and 12 years after 
[Project Panama](https://openjdk.org/projects/panama/) was introduced, Vector APIs are still an incubating 
feature in Java. In addition, while I haven't experimented with them myself, there are 
[some indications](https://bluuewhale.github.io/posts/further-optimizing-my-java-swiss-table/) that even 
the current incubating design is still slower than using bit-twiddling SWAR (SIMD In A Register) techniques 
when it comes to hashtable usage.

Interestingly, for GetHit performance AndroidX is only competitive in the L3 cache regime. When the table 
either fits in L1/L2 or is larger than L3, AndroidX is substantially slower than other libraries. I do not 
have a good hypothesis to explain this behavior.

In addition, AndroidX has some interesting failure cases (highBits keys, and performance cliff at ~32M entries) - 
but it's perhaps unlikely these would cause real world issues for most usages (who's loading 32M entries into a 
map on Android?).

---

#### Eclipse

Eclipse's performance appeared exceptional, but as we investigated it became clear that this was entirely 
due to using a much lower maximum load factor than any other library. Once this was accounted for, Eclipse 
was reasonably in the middle of the pack.

I was very interested to see how Eclipse's multiple probing strategies (3 different hash functions) would work out. 
The idea of first attempting the identity hash for half a cache line before switching to a more rigorous 
finalizer seems like it could pay dividends, but at least for Eclipse it does not appear to have been a game 
changer.

---

#### Fastutil/HPPC/Koloboke/Agrona

All of these libraries implement pretty much the same hashtable design (linear probing, same hash finalizer, same 
memory layouts), with only minor variations that lead to small differences in performance.

Within this family of tables, HPPC tends to generally be the fastest and Agrona the slowest. This appears to be 
more due to smaller differences in implementation rather than anything algorithmic.

---

#### Trove

The only prime-sized table here - it's good to see and evaluate a different approach, but overall it couldn't 
really compete with the other libraries, and was frequently even outperformed by the JRE HashMap. Trove did perform 
exceptionally well with lowBits keys due to using the identity hash finalizer, but if we're specializing for
lowBits keys I suspect other table designs would out-perform Trove by using the identity hash as well. Trove's  
general lack of performance appears to have been noted as early as 2017[^1].

[^1]: Diego Costa, Artur Andrzejak, Janos Seboek, and David Lo. 2017. Empirical Study of Usage and Performance of 
Java Collections. In Proceedings of the 8th ACM/SPEC on International Conference on Performance Engineering (ICPE '17).
https://doi.org/10.1145/3030207.3030221

---

In the end, performance across all these libraries is respectable. The larger benefit is memory savings - the
secondary benefit is CPU savings. And while it's quite entertaining to squeeze every bit of performance possible out of
these tables, the vast majority of code written in the JVM ecosystem is unlikely to care.

## All Results

At this point, the only thing left is all the raw benchmarking results - feel free to continue scrolling if those 
are of interest, otherwise there's nothing more to read.

### Int → Int / getHit

{{< benchmark-chart benchmark="IntMap.getHit" order="random" title="Int → Int / getHit — random keys" >}}
{{< benchmark-chart benchmark="IntMap.getHit" order="lowBits" title="Int → Int / getHit — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.getHit" order="highBits" title="Int → Int / getHit — highBits keys" >}}

### Int → Int / getMiss

{{< benchmark-chart benchmark="IntMap.getMiss" order="random" title="Int → Int / getMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.getMiss" order="lowBits" title="Int → Int / getMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.getMiss" order="highBits" title="Int → Int / getMiss — highBits keys" >}}

### Int → Int / putHit

{{< benchmark-chart benchmark="IntMap.putHit" order="random" title="Int → Int / putHit — random keys" >}}
{{< benchmark-chart benchmark="IntMap.putHit" order="lowBits" title="Int → Int / putHit — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.putHit" order="highBits" title="Int → Int / putHit — highBits keys" >}}

### Int → Int / putMiss

{{< benchmark-chart benchmark="IntMap.putMiss" order="random" title="Int → Int / putMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.putMiss" order="lowBits" title="Int → Int / putMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.putMiss" order="highBits" title="Int → Int / putMiss — highBits keys" >}}

### Int → Int / removeAndPutMiss

{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="random" title="Int → Int / removeAndPutMiss — random keys" >}}
{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="lowBits" title="Int → Int / removeAndPutMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="IntMap.removeAndPutMiss" order="highBits" title="Int → Int / removeAndPutMiss — highBits keys" >}}

### Int → Int / forEach

{{< benchmark-chart benchmark="IntMap.forEach" title="Int → Int / forEach" >}}

### Int → Int / naiveCopy + preAllocatedCopy

{{< benchmark-chart benchmark="IntMap.naiveCopy" title="Int → Int / naiveCopy" >}}
{{< benchmark-chart benchmark="IntMap.preAllocatedCopy" title="Int → Int / preAllocatedCopy" >}}

### Long → Int / getHit

{{< benchmark-chart benchmark="LongMap.getHit" order="random" title="Long → Int / getHit — random keys" >}}
{{< benchmark-chart benchmark="LongMap.getHit" order="lowBits" title="Long → Int / getHit — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.getHit" order="highBits" title="Long → Int / getHit — highBits keys" >}}

### Long → Int / getMiss

{{< benchmark-chart benchmark="LongMap.getMiss" order="random" title="Long → Int / getMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.getMiss" order="lowBits" title="Long → Int / getMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.getMiss" order="highBits" title="Long → Int / getMiss — highBits keys" >}}

### Long → Int / putHit

{{< benchmark-chart benchmark="LongMap.putHit" order="random" title="Long → Int / putHit — random keys" >}}
{{< benchmark-chart benchmark="LongMap.putHit" order="lowBits" title="Long → Int / putHit — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.putHit" order="highBits" title="Long → Int / putHit — highBits keys" >}}

### Long → Int / putMiss

{{< benchmark-chart benchmark="LongMap.putMiss" order="random" title="Long → Int / putMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.putMiss" order="lowBits" title="Long → Int / putMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.putMiss" order="highBits" title="Long → Int / putMiss — highBits keys" >}}

### Long → Int / removeAndPutMiss

{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="random" title="Long → Int / removeAndPutMiss — random keys" >}}
{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="lowBits" title="Long → Int / removeAndPutMiss — lowBits keys" >}}
{{< benchmark-chart benchmark="LongMap.removeAndPutMiss" order="highBits" title="Long → Int / removeAndPutMiss — highBits keys" >}}

### Long → Int / forEach

{{< benchmark-chart benchmark="LongMap.forEach" title="Long → Int / forEach" >}}

### Long → Int / naiveCopy + preAllocatedCopy

{{< benchmark-chart benchmark="LongMap.naiveCopy" title="Long → Int / naiveCopy" >}}
{{< benchmark-chart benchmark="LongMap.preAllocatedCopy" title="Long → Int / preAllocatedCopy" >}}
