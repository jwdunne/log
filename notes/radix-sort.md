# Radix sort

Radix sort is a non-comparison sort. It sorts, instead, by sorting items into buckets, according to their _radix_.

For items with more than one significant digit, the process of sorting into buckets is repeated, until all digits have been considered.

A digit can be the digits of an integer. Or they could be the byte of a string.

## Why this matters

This means the worst case could be a lot better than the theoretical upper limit of what can be achieved with a comparison sort i.e `O(n x m)` where $m$ is the length of digit vs `O(n log n)`.

In practice, for collections of strings and integers, this can be between 50% to three times faster than general-purpose sorting implementations.

## Implementation

A simple Rust implementation using counting for bytes in a slice of bytes is given below:

```rust
fn sort_by_first_byte(items: &mut [u8], output &mut [u8]) {
  let mut counts = [0u32; 256];
  for item in items {
    counts[byte] += 1;
  }

  let mut offsets = [0u32; 256];
  let mut total = 0;
  for i in 0..256 {
    offsets[i] = total;
    total += counts[i];
  }

  for item in items.iter() {
    let pos = offsets[item] as usize;
    output[pos] = *item;
    offsets[item] += 1;
  }
}
```

This is quite naive and can be improved but demonstrates the simple idea. There is a variant that sorts in place but this is apparently quite complex.

It could also be extended to any data type with generics.
