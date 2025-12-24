# 2025-12-21: Custom hash table implementation

I implemented and optimised a custom hash table implementation for the 1 billion row challenge. 

In my previous attempts, I was using `hashbrown::HashMap`, which is what Rust uses under the hood but provided a convenient `HashMap::entry_ref()` API. This allowed us to avoid allocations when looking up keys when using `entry`, which was a problem with the standard library implementation.

## The design

- Simple vec of `Entry` structs, using linear probing, for better cache locality
- Keyed via simple hash function, good enough for our purposes, to lower constant cost
- Small API designed around minimising work in the hot loop (i.e lookup returns a slot number)

## Initial results

On first implementation, it sucked. It was about twice as slow as `hashbrown::HashMap`.

Over a few days, I cut this down. The first problem was cache locality. This is naturally challenging for any hash map implementation, but I'd made it worse without due care towards alignment of data in `Entry`.

Things that helped:

- Tightly controlling the `Entry` layout
- Separating station names from `Entry` values entirely
- Introducing a `prefix` as a `u64` for fast comparison of the first 8 bytes
- Hard coding the maximum identified probe depth (5)
- Using hash and prefix to identify entries (rather than comparing names) on conflict
- Improving the hash function to work on only the prefix when 8 bytes or less
- Improving the hash function further by working on only the prefix and suffix with 16 bytes or less
- Computing the prefix once when hashing and returning that, instead of repeating the work

## Further improvements

Around other performance improvements, I further optimised the hash table implementation:

- Made the lookup branchless given the consistent probe depth
- Moved hashes and prefixes to their own arrays to enable SIMD lookup using AVX512 intrinsics
- Folded the up to 8 byte and 16 byte special cases in the hash function into one to ease branch mispredicts
- Replaced prefix computation with a branchless solution
- Extend API to support prefetching slots from computed hashes
