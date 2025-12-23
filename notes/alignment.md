# Alignment

Memory is byte-addressable. But the CPU doesn't fetch memory one byte at a time. Instead, memory is read through the bus, which is typically 64 bits wide (8 bytes).

A value in memory is _aligned_ when its address is a multiple of its size. That is, a `u64` is aligned when it sits at an address that is a multiple of 8. And a `u32` is aligned when it sits at an address that is a multiple of 4.

## Why this matters

When a value is aligned, the CPU needs only one fetch from memory to grab it, which is the optimum.

When a value is unaligned, however, it straddles the boundary. The CPU requires 2 fetches from memory to read the data. On top of this, the CPU then needs to extract the relevant bits and reassemble them. This adds significant overhead.

## Alignment and SIMD

SIMD registers are wider i.e 128, 256 and 512 bits, which magnifies the alignment problem. Loading a 256-bit value from an address not divisible by 32 means a significant amount more work.

You must also make a conscious choice. `_mm_load_si128` loads aligned values into the registers, where as `_mm_loadu_si128` loads unaligned values, the latter being significantly slower.

## In practice

The CPU prefers data to reside in memory aligned. Respecting alignment means single-fetch memory access and enables stronger optimisations. Ignoring it means slower memory access, hardware faults or even undefined behaviour depending on the platform and how data is read.
