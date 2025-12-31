# What every programmer should know about memory

- Most often/likely used data kept in main memory
- Removing main memory as a bottleneck is _hard_
- Comes from these areas:
    * RAM hardware design
    * Memory controller designs
    * CPU caches
    * Direct memory access (DMA) for devices

## 2. Commodity hardware today

Modern, standardised chipset layout:

- Standardised on 2-part chipset: northbridge and southbridge
- CPUs connected via common bus (Front Side Bus, FSB) to northbridge
- Northbridge contains memory controller 
- Memory controller implementation determines type of RAM supported
- Northbridge must communicate with Southbridge to reach all other system devices
- Southbridge often called I/O bridge
- PCI-E slots are all connected to the Southbridge

Consequences of this design:

- Communication between CPUs must use same bus to communicate with Northbridge
- All comms with RAM must pass through Northbridge
- RAM has only a single port
- Comms from CPU to devices on Southbridge must pass through Northbridge

Immediately apparent bottlenecks:

- Access to RAM for devices using DMA increases contention for bandwidth of Northbridge
- DMA requests compete with RAM access from CPUs
- Bandwidth of bus from Northbridge to RAM is second bottleneck
- Processors must wait to access memory
- Wait times increase with simultaneous access to memory from hyper-threads/cores/processors
- Applies to DMA operations 
- Access patterns are a massive influence too

Some expensive systems seek to work around this:

- Employs multiple memory controllers, meaning multiple buses, 
- Total available bandwidth is increased
- Total supported memory is also increased
- Concurrent memory access patterns reduce delays
- But primarily limited by internal bandwidth of Northbridge

Integrating memory controllers into the CPUs is another approach:

- As many memory banks available as there are processors
- On a quad-CPU machine, memory bandwidth is quadrupled 
- W/o complicated Northbridge with enormous bandwidth

Disadvantages of CPU-integrated memory controllers:

- Memory must still be accessible to all processors
- But memory is no longer uniform 
- Hence NUMA - Non-Uniform Memory Architecture
- Processor-local memory accessible at usual speed
- Access to another CPU's memory is _complicated_
- Memory of adjacent CPU requires crossing a single interconnect
- Non-adjacent, two interconnects (assuming quad-CPU design)
- Number of levels grows significantly with more complicated machines
- Hence cost of communication (NUMA factors) and can be quite high

### 2.1 RAM Types

- There exists both SRAM (static RAM) and dynamic RAM (DRAM)
- SRAM is significantly faster than DRAM
- SRAM is much more expensive to produce and use than DRAM

#### 2.1.1 SRAM

- Formed by four transistors, M1 to M4
- Form two cross-coupled inverters
- Two stable states: 0 and 1 respectively
- State is stable with power available on Vdd
- WL is the word-access line
- WL is raised if cell state required
- Cell immediately available for reading on BL and BL'
- To overwrite, BL and BL' first set to desired value then WL is raised

Important to note that:

- One cell = six transistors
- Cell state requires constant power
- Cell state is available immediately once WL is raised
- Cell state is stable, _no refresh cycles required_

#### 2.1.2 DRAM

- Much simpler structure than SRAM
- Consists of one transistor and one capacitor
- State kept in capacity C
- Transistor M guards state access
- Access line AL is raised to read state
- Charge of capacitor determines state
- Current or lack thereof on data line DL is state
- To set state, 
    * DL is appropriately set and 
    * AL is raised long enough to charge capacitor

Complications:

- Reading state discharges capacitor 
- Capacitor must be recharged
- Capacity of capacitor must be _low_
- Fully charged capacitor holds a few 10s of thousands of electrons
- Charge dissipation happens rapidly (leakage)

Problem 1, constant refresh cycles:

- Constant refresh cycles required
- Modern DRAM chips require once every 64ms
- During refresh, memory access is blocked
- Stalls up to 50% of memory access for some workloads

Problem 2, not directly usable:

- Data line must be connected to sense amplifier
- Distinguishes between stored 0 or 1 over range of charges counting as 1

Problem 3, reading depletes capacitor:

- Every read must be followed by a capacitor recharge
- Reading memory content requires additional energy
- And, more importantly, time

Problem 4, charge/drain not instant:

- Signals received by sense amplifier _not_ rectangular
- Takes time, depending on capacity C and resistance R
- Severely limits how fast DRAM can be

Advantages of simple approach:

- Size: a DRAM cell is many times smaller than an SRAM cell
- Power: SRAM cells need individual power to maintain state
- Regularity: DRAM cells can be packed tighter than SRAM cells

Dramatic difference in cost wins. Thus, we live with DRAM, leading
to huge implications on programmer and the programs they write.

#### 2.1.3 DRAM Access

- Program selects memory locations using virtual addresses
- Processor translates to physical addresses
- Memory controller selects RAM corresponding to address
- Parts of the physical address are passed in the form of address lines

Individual memory addressing is impractical - 4GB of RAM would require
2^32 address lines.

Instead, address must be demultiplexed - demultiplexer with N address 
lines will have 2^N output lines. Output lines used to select memory
cell. No problem for small capacity chips. For a larger number of cells,
however:

1. Size/complexity of demultiplexer increases exponentially
2. Requires a tonne of chip real estate
3. 30 impulses on address lines synchronously is much harder than just 15
4. Must be laid out exactly the same length or timed appropriately

Instead, DRAM cells are organised in rows and columns:

- Can get by with a single demultiplexer
- And a single multiplexer 
- Both (equivalent from now on) need only be half the size

Reading:

1. Rows selected via address lines a0 and a1 through _row address selection_ (RAS) demultiplexer 
2. Content of all cells made available to column address selection (CAS) multiplexer on address lines a2, a3

Writing:

1. New cell put on data bus
2. Cell is selected using RAS and CAS
3. Data stored in CELL

#### 2.1.4 Conclusions

- good reasons why not all memory is SRAM
- memory cells need to be individual selected to be used
- number of address lines is directly responsible for cost of:
    * memory controller
    * motherboards
    * DRAM module
    * DRAM chip
- takes a while before read/write is available

### 2.2 DRAM Access Technical Details

- SDRAM: synchronous DRAM
- DDR: Double Data Rate DRAM
- Memory controller provides clock
- Frequency determines speed of Front Side Bus (FSB)
- SDRAM data transfer: 64 bits, 8 bytes
- Transfer rate of FSB: 8 bytes by effective bus frequency
- E.g 6.4GB/s for quad-pumped 200MHz bus
- Burst speed, not effective speed - there is a lot of downtime
- Minimising this downtime achieves the best performance

#### 2.2.1 Read access protocol

- Read cycle begins with memory controller: 
    * Making row address available on address bus
    * Lowering RAS signal
- All signals read on rising edge of the clock (CLK)
- Setting row address causes RAM chip to start latching addressed row
- CAS signal can be sent after tRCD (RAS-to-CAS delay) clock cycles
- Column address made available on address bus, lowering the CAS line
- Addressing complete, data can be transmitted
- RAM chip requires prep time: the CAS latency (CL)
- With CL=2.5, first dataavailable after first falling flank in blue area

For maximum utility of prep work, DRAM allows mem controller to specify
how much data is to be transmitted (often, 2, 4, or 8 words). This allows:

1. Filling entire lines in the cahces without a new RAS/CAS sequence
2. Sending new CAS signal w/o resetting row selection

Consecutive memory addresses can be read from or written to significantly
faster because the RAS signal does not need to be sent, and the row does
not need to be deactivated.

#### 2.2.2 Precharge and activation

Before a new RAS signal can be sent:

- Current latched row must be deactivated
- New row must be precharged

Precharge cannot be issued right away - must wait as long as it takes
to transmit data (2 cycles in the example).

Precharge takes tRP (Row Precharge time) cycles until row can be selected.

In the diagram, precharge overlaps with memory transfer, which is good. 
But tRP is larger than the transfer time, so the next RAS signal is delayed
by 1 cycle.

Continuing this way, data transfer happens 5 cycles after the previous one
stops, which is bad (data bus only used in 2 cycles out of 7).

DDR modules are often described using special notation:

| Symbol | Value | Meaning      |
| ------ | ----- | ------------ |
| w      | 2     | CL           |
| x      | 3     | tRCD         |
| y      | 2     | tRP          |
| z      | 8     | tRAS         |
| T      | T1    | Command rate |

#### 2.2.3 Recharging

- Study found DRAM refresh organisation can affect performance dramatically 
- Each DRAM cell must be refreshed every 64ms according to JEDEC
- With 8,192 rows, mem ctrlr must refresh every 7.812 microseconds
- Memory controller responsible for scheduling refreshes
- DRAM maintains address of last refreshed row
- DRAM auto-increments address counter for each request

#### 2.2.4. Memory types

- SDRAM: memory cell/data transfer rate identical
- DDR: transports data on rising _and_ failing edge

Main effects of using serial bus:

1. More modules per channel can be used
2. More channels per Northbridge/memory controller can be used
3. Serial bus is designed to be fully-duplex (two lines)
4. Cheap enough to implement differential bus and increase the speed

#### 2.2.5 Conclusions

- DRAM access _is not fast_ relative to processor and register/cache access
- E.g Intel Core 2 running 2.9GHz with FSB 1GHz, clock ratio: 11:1
- Each stall of 1 cycle of memory bus: 11 CPU cycles
- On most machines, this is actually slower so delays are greater
- HW/SW prefetching can create more overlap in timing, reducing the stall
- Prefetching also shift memory ops in time, meaning less contention later
- By shifting read in time, write and reads do not have be issued simultaneously

## 3. CPU caches

- CPU caches are small blocks of SRAM
- Administered directly by the CPU
- Used to _cache_ data from main memory likely to be used soon
- Program code and data has temporal and spatial locality

Data not close together in memory but accessed recently together
is likely to be accessed again, which is temporal locality.

This applies to code too - think of a function, stored elsewhere
in memory, called repeatedly in a loop.

Loops are also a good example of spatial locality - the same 
instructions executed repeatedly.

Effectiveness of caches:

- Main memory access: 200 cycles
- Cache memory access: 15 cycles
- Code using 100 data elements 100 times
- 2,000,000 cycles on mem ops without cache
- 168,500 cycles if all data can be cached
- 91.5% improvement

SRAM is much smaller than main memory: e.g 4MB cache, 4GB main memory

### 3.1 CPU caches: the big picture

Mental model:

- All loads/stores go through cache
- Connection between CPU/cache is fast
- Cache/main mem connected via FSB
- In spite of von neumann arch, separate cache for code/data
- Why? code/data pretty much independent regions 
- Also: instruction decoding is slow and caching helps

After intro of cache system:

- Speed delta between cache/main mem increased
- So another level of cache was added
- Bigger and slower than first-level cache

Today 3 levels are common:

- L1d: level 1 data cache
- L1i: level 1 instruction cache
- L2 cache
- L3 cache

Example layout:

- 2 threads per core sharing L1 cache
- 2 cores, separate L1 caches per core, sharing L2/L3 caches
- 2 processors, separate L2 and L3 caches per processor

### 3.2 High-level cache operations

- All data read or written by CPU cores is cached
- Regions of uncacheable main memory exist, but OS-only concern
- Instructions exist to bypass caches
- If CPU needs word, caches searched first
- Each cache entry tagged with main memory address
- Inefficient to chose a word for cache granularity
- Remember: more effective if many data transferred without new CAS/RAS signal
- Entries stored in caches are not single words, but lines
- Early lines were 32 bytes long
- 64 bytes is now the norm
- Memory bus = 64 bits wide, 8 transfers per cache line

Memory address for each cache line is computed by masking the address value
according to the cache line size e.g for a 64 byte cache line, low 6 bits zeroed.

Discarded bits are used as offset into cache line. Remaining bits can be used
to locate the line in cache and as the tag.

Instruction modifies memory, processor must still load a cache line first - no
instruction modifies an entire cache line at once (except write-combining, which
I suspect means SIMD?)

Eviction from L1d pushes the cache line down to L2, using the same cache line size.

Eviction from L2 pushes data into L3. Eventually, eviction out of the cache entirely.

Each eviction is progressively more expensive - this model is called the exclusive cache,
favoured by AMD/VIA processors. Intell implements inclusive caches where each cache line in
L1d exists in L2, so eviction from L1d is faster (at the expense of hydration, which is minimal)

Costs of cache hits and misses (for Intel Pentium M):

- Register: 1 cycle or less
- L1d: ~3 cycles
- L2: ~14 cycles
- Main memory: ~240 cycles

These look high (and they are), but pipelining can help if load instructions are issued early enough
such that they can be performed in parallel.

### 3.3 Cache Implementation Details

_(to be read another day with more practice)_

### 3.4 Instruction cache

- Instructions are cached as well as data
- Quantity of code executed depends on size of code needed
- Size of code depends on complexity of problem
- Complexity of problem is _fixed_
- Program's instructions generated by a compiler
- Compiler writes optimise for well-generated code
- Program flow is far more predictable than data access
- Code always has quite good spatial and temporal locality

The instruction pipeline:

1. Instruction is decoded
2. Parameters are prepared
3. Instruction is executed

Pipelines:

- Can be quite long (maybe 20 plus steps)
- If it stalls, takes a while to get up to speed again
- Pipeline stalls happen if next instruction cannot be predicted
- Or takes too long to load the next instruction (when read from memory)
- Branch prediction minimises pipeline stalls
- On CISC, decoding can take some time
- Instead, they cache the decode instructions
- Called the trace cache
- Trace caching allows skipping of first steps of pipeline
- L2 are unified, containing both code and data
- Code in L2 is _not_ cached decoded

Rules for best performance re. instruction cache:

1. Generate code as small as possible
2. Help the processor make good prefetching decisions via code layout or explicit prefetching

Self-modifying code should be avoided because it can create performance problems, naturally.

### 3.5 Cache miss factors

#### 3.5.1 Cache and memory bandwidth

For 64-bit Intel Netburst processor:

- Working set fits into L1d, reads 16B per cycle
- Working set larger than L1d, reads 6B per cycle
- Write and copy never exceeds 4B per cycle
- Intel used write-through mode, so L1d write is limited by L2
- Write performance drops to 0.5B per cycle once L2 is insufficient
- Optimising writes is even more important for performance

For two threads, one for each hyper thread:

- Each thread only has half the cache/bandwidth available
- Each thread spend a lot of time waiting for memory
- Shows worst possible use of hyper-threads

Core 2 processor (1 thread):

- Write/copy for L1d is comparable to reads
- Once L1d is insufficient, performance cliff dives
- With 2 threads, write performance is same as if data had to be read from main mem
- Read performance isn't, however, impacted

#### 3.5.2 Critical word load

- Memory transferred from main mem into cached in blocks smaller than the cacheline
- Today, 64 bits are transferred at once
- Cache line size is 64 or 128 _bytes_
- 8 or 16 transfers per cache line required
- DRAM chips can transfer 64-byte blocks in burst mode
- Can fill cache line w/o further commands from mem controller
- Can be up to 30 cycles before getting the data word required
- Doesn't have to be like this - process can communicate _critical word_
- Memory controller can request this word first
- Once the word arrives, program continues whilst cache line arrives
- If processor prefetches, critical word is not known

#### 3.5.3 Cache placement

- Not under direct control of programmer
- Programmers _can_ determine where thread are executed
- Hyper-threads share everything inc. L1 caches
- Each core has it's own L1 cache
- Not much more in common beyond that
- Intel have shared L2 caches, no higher level caches
- AMD have separate L2 caches, unified L3 cache
- Future cache design will involve more layers-
- Programmers must know workloads and details of machine architecture for best performance

#### 3.5.4 FSB influence

- Cache content can only be stored/loaded as quickly as the bus allows
- Intel processes support FSB speeds up to 1,333MHz
- Higher speeds will come in future
- If working set sizes are larger, fast RAM and high FSB is worth investment

## 4. Virtual memory

_(to be read later)_

## 5. NUMA support

_(to be read later)_

## 6. What programmers can do 

Covering:

- Physical RAM access
- L1 caches
- OS functionality

### 6.1 Bypassing the cache

- Writes that do nothing with data later harm performance
- Full cache line read first and cached data modified
- Pushes more frequently accessed data out of caches 
- Especially true for large matrices - filled but used later
- Processors provide non-temporal write operations
- Does not need to be expensive
- Processor will use write-combining to fill cache lines

x86/x86-64 intrinsics:

```
#include <emmintrin.h>
void _mm_stream_si32(int *p, int a);
void _mm_steeam_si128(int *p, __m128i a);
void _mm_stream_pd(double *p, __m128d a);

#include <xmmintrin.h>
void _mm_stream_pi(__m64 *p, __m64 a);
void _mm_stream_ps(float *p, __m128 a);

#include <ammintrin.h>
void _mm_stream_sd(double *p, __m128d a);
void _mm_stream_ss(float *p, __m128 a);
```

About the intrinsics:

- Most efficient when processing large amount of data in one go
- Data loaded, processed in or more steps, and written back
- Data "streams" hence naming
- Memory address must aligned to 8 or 16 bytes
- Possible to replace normal \_mm_store with these non-temporal versions
- Must modify single cache line one after another for write-combining

```
#include <emmitrin.h>

void setbytes(char *p, int c)
{
    __m128i i = _mm_set_epi8(
        c, c, c, c,
        c, c, c, c,
        c, c, c, c,
        c, c, c, c
    );

    _mm_stream_si128((__m128i *)&p[0], i);
    _mm_stream_si128((__m128i *)&p[16], i);
    _mm_stream_si128((__m128i *)&p[32], i);
    _mm_stream_si128((__m128i *)&p[48], i);
}
```

This code:

- If pointer `p` is aligned, this fn will set all bytes to `c`
- Avoids reading the cache line before it is written
- Avoids polluting the cache with data which might not be needed soon
- Example: C's `memset`, which writes a large blocks of memory in one go
- PowerPC arch defines `dcbz` instruction clears an entire cache line

Initialising a matrix:

- Incrementing row inner loop:
    * Normal: 0.048s
    * Non-temporal: 0.048s
- Incrementing col inner loop:
    * Normal: 0.127s
    * Non-temporal: 0.160s

Col-wise access is slower than cached access because no write-combining
is possible and each memory cell must be addressed individually.

Intel with SSE4.1 extensions introduced NTA (non-temporal access)
loads. They are accessed using a small number of streaming load buffers,
each buffer containing a cache line. An intrinsic is given for this:

```
#include <smmintrin.h>
__m128i _mm_stream_load_si128(__m128i *p);
```

Notes:

- Should be used multiple times
- With addresses of 16-byte blocks as parameters
- Up until each cache line is read
- Might be possible to read from two memory locations at once

### 6.2 Cache access

- Focus first on optimising for L1 cache
- Theme:
    * Improve locality (spatial and temporal)
    * Align code and data

#### 6.2.1 Optimising L1 data cache access

Example, matrix multiplication of 1000 x 1000 `double` elements:

```c
for (i = 0; i < N; ++i)
    for (j = 0; j < N; ++j)
        for (k = 0; k < N; ++k)
            res[i][j] += mul1[i][k] * mul2[k][j];
```

- `mul1` is accessed sequentially
- Inner loop advances row number of `mul2`
- `mul1` row-incrementing inner loop
- `mul2` col-incrementing inner loop - significantly slower
- Possible remedy: transposing `mul2`

With transposition:

```c
double tmp[N][N];
for (i = 0; i < N; ++i)
    for (j = 0; j < N; ++j)
        tmp[i][j] = mul2[j][i];

for (i = 0; i < N; ++i)
    for (j = 0; j < N; ++j)
        for (k = 0; k < N; ++k)
            res[i][j] += mul1[i][k] * tmp[j][k];
```

Comparison on Intel Core 2 with 2.6GHz clock:

- Original:   16,765,297,870 cycles
- Transposed:  3,922,373,010 cycles (23.4%)
- 76.6% speed up
- But can still be improved

Examining the hardware:

- Each round of inner loop requires 1000 cache lines
- Way more than the 32k of L1d available
- In this example, using two `double`s halves L1d miss rate 
- Core 2 has L1d cache line size of 64 bytes
- `sizeof(double)` is 8 bytes
- We should unroll middle loop 8 times to fully use cache line
- Unroll outer loop 8 times to write 8 results at a time
- It's best to hardcode cacheline sizes at compile time using `getconf`

Resultant code:

```c
#define SM (CLS / sizeof (double))

for (i = 0; i < N; i += SM)
    for (j = 0; j < N; j += SM)
        for (k = 0; k < N; k += SM)
            for (
                i2 = 0, rres = &res[i][j], rmul1 = &mul1[i][k];
                i2 < SM;
                ++i2, rres += N, rmul1 += N
            )
                for (
                    k2 = 0, rmul2 = &mul2[k][j];
                    k2 < SM;
                    ++k2, rmul2 += N
                )
                    for (j2 = 0; j2 < SM; ++j2)
                        rres[j2] += rmul1[k2] * rmul2[j2];
```

Notes:

- Six nested loops
- Outer loops iterate in intervals of `SM` (cache line size over `sizeof (double)`)
- Divides multiplication in several smaller problems with better cache locality
- 3 inner loops iterative over missing indexes of outer loops
- `k2` and `j2` are in a different order
- Why? Only 1 expression depends on `k2` but two depend on `j2`
- Complication results to appease gcc and optimisation of array indexing
- Introduction of `rres`, `rmul1` and `rmul2` pulls common expressions out of inner loops
- Unless `restrict` is used, all pointer accesses are potential aliasing sources

> Note: what is meant by aliasing here?
> 
> Compiler cannot safely hoist into a register in case of overlapping memory
> i.e because a change to the accumulator may change what is read on next iteration,
> which isn't actually possible here (but the compiler doesn't know that)
>
> By manually hoisting into a reference as high as possible, the compiler _knows_
> it lives in a register and cannot alias with anything, allowing optimisation.

Results:

- Original:   16,765,297,870 cycles
- Transposed:  3,922,373,010 cycles (23.4%)
- Sub-matrix:  2,895,041,480 cycles (17.3%)
- Vectorised:  1,588,711,750 cycles (9.47%)

For this source code:

```c
struct foo {
    int a;
    long fill[7];
    int b;
}
```

`pahole` shows:

- shows data structure size (72 bytes)
- shows it uses more than one cache line
- shows sizes of each member
- shows a 4 bytes hole (suggesting packing)
- in this example, packing is simple
- we move `int b` up 
- this aligns with cache line

Programmers should follow two rukles:

1. Move the structure element which is most likely to be 
   the critical word to beginning of structure
2. Access elements in the order in which they are defined
   in the structure
3. For small structures, arrange elements in order they
   are likely accessed
4. For bigger structures, each cache line-sized block
   should be arranged to follow the rules

Notes on alignment:

- Don't rearrange if it means _alignment_ is lost
- If object isn't aligned as expected, reordering isn't worth it
- Each fundamental type has it's own alignment requirement
- For structured types, the largest alignment requirement of any
  of its elements determines the alignment of the structure
- This is almost always smaller than the cache line size
- Even if members of a structure are lined up to fit into the same
  cache line, they may not have an alignment matching the cache line size

Two ways to ensure the object has the alignment which was used when
designing the layout of the structure:

- Allocate object with explicit alignment requirement:
    * For dynamic allocation, a call to `malloc` would only allocate
      the object with an alignment matching of the most demanding type
    * Alignment requirement can be changed of a user-define type
      using a type attribute


Multi-media extensions almost always require aligned memory access i.e
16 byte memory accesses the address is supposed to be 16 byte aligned.

Efficient placement of structure elements and alignment are not the
only thing that influence cache efficiency e.g with an array of 
structures, the entire structure def affects performance.

Example:

```c
struct order {
    double price;
    bool paid;
    const char *buyer[5];
    long buyer_id;
}
```

If stored in a big array and a frequently-run job adds up expected
payments of all outstanding bills, `buyer` and `buyer_id` field
unnecessarily load into caches. The program performs up to 5 times
worse than it could.

In this case, it's better to split the order data structure into two,
storing the first two fields in one structure and other fields elsewhere.

#### 6.2.2 Optimising L1 instruction cache access

- Preparing code for L1i use is similar to L1d use
- Except programmer does not directly influence this
- Unless writing code in assembler
- However, compilers can be indirectly guided towards better layout

Code is linear between jumps - the processor can prefetch efficiently.
But jumps disrupt this:

- Jump dest might not be statically determined
- Even if static, it might miss all caches, and take longer

To achieve best L1i use:

1. Reduce code footprint as much as possible, balanced against
   loop unrolling and inlining
2. Code execution should be linear without "bubbles"
3. Aligning code when it makes sense

> Bubbles describe visually the holes in the execution in the
> pipeline of a processor which appear when execution has to
> await resources.

Inlining always makes sense when a function is only called once, 
giving more optimisation opportunities.

Another big advantage of small loops: intel core 2 frontend has
a special feature called Loop Stream Detector (LSD). If a loop has
no more than 18 instructions, none of which is a subroutine call,
requires only up to 4 decoder fetches of 16 bytes, has at most 4
branch instructions and is executed more than 64 times, the loop is
sometimes locked in the instruction queue and therefore more 
quickly available when the loop is used again. This applied to small
inner loops which are entered many times through an outer loop.

Alignment is important in code too, though far more difficult to
control. Alignment of code is most useful:

- at the beginning of functions
- at the beginning of basic blocks which are reached only via jumps
- to some extent, at the beginning of loops

Three ways a programmer can influence code alignment:

1. If the code is written in assembler, it can be explicitly aligned
2. For compiled languages, a compiler option is used i.e `-falign-functions=N`
3. And for loops `-falign-loops=N`

#### 6.2.3 Optimising L2 and higher cache access

- Everything said about optimisation for L1 cache applies to L2 and beyond
- Except cache misses are _always_ expensive
- L1 misses hopefully frequently hit L2 and higher, limiting penalties
- There is no fallback for the last level of cache
- L2 or higher level caches are often shared by multiple cores
- To avoid high cache miss penalty, working set size should match cache size

L1 cache line size is usually constant over processor generations. As such,
it's reasonable to hardcode the cache line size and optimise code for it.

For higher level caches, this is not the case if the program is supposed to
be generic - sizes can vary widely (factors of 8 or more are common).

It's not possible to assume larger cache sizes as a default, since performance
will suck on everything but the machines with the biggest LL cache line sizes.

Same too with assuming the smallest - it'd throw away performance for all but
the smallest cache lines.

Instead, code must dynamically adjust itself to the cache line size:

1. Program finds `/sys/devices/system/cpu/cpu*/cache`
2. Identify last level cache from highest numeric value in `level` file
3. Read content of `size` file in that directory
4. Divide numeric value by number of bits set in bitmask in file `shared_cpu_map`

Result is a safe lower limit.

#### 6.2.4 Optimising TLB usage

Two kinds of TLB usage optimisation:

1. Reduce number of pages a program has to use, automatically resulting in
   fewer TLB misses
2. Make TLB lookup cheaper by reducing the number of higher level directory
   tables which must be allocated - fewer tables, less memory used, higher
   cache hit rates.

> TLB is the Translation Lookaside Buffer. An extremely small, extremely
> fast cache inside the CPU that stores recent virtual-to-physical address
> translations.
>
> It exists to reduce cost of virtual address lookups. Walking the page
> table is expensive, which is required to translate. The TLB caches
> these translations so subsequent accesses to the same page are nearly
> free.

### 6.3 Prefetching

### 6.4 Multi-thread optimisations

### 6.5 NUMA programming
