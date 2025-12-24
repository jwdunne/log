# 2025-12-22: SIMD line parsing

## What I did

Improve line parsing on my 1BRC solution by using AVX2 to search 32 bytes of delimiters in parallel. 

## What I learned

- Basic AVX2 intrinsics in Rust e.g `_m256_loadu_si256` to load unaligned bytes
- Using `cfg` macro to swap out implementations at compile time based on feature flags

## What I plan next

- Using AVX512 to check 64 bytes at a time
- Using SIMD to improve linear probing performance
- Batching to encourage instruction-level parallelism
- Parallelism across threads
