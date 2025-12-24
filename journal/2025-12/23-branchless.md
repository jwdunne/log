# 2025-12-23: Cost of branching

## What I did

- Enabled branch predictor simulation in callgrind
- Packed bytes into a `u64` without branching 
- Parsing temperatures between -99.9 and 99.9 without branching

## What I learned

- How impactful branching and branch mispredictions can be in hot loops
- How manipulating bits can eliminate branches in hot loops

## What next

- Interleaving the same operations on independent data for ILP
- Memory-mapping files
- Use AVX512 to widen parallel delimiter search
- Use AVX512 to improve probing performance
