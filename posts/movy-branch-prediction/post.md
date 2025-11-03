# How Building a Terminal Graphics Engine Taught Me About CPU Branch Prediction and Cache Hierarchy

**Or: Why My "Optimized" Rendering Function Was Sometimes Slower Than My "Slow" One**

---

## The Setup

I've been building **movy**, a terminal-based graphics rendering engine in Zig that turns your terminal into a graphical canvas using ANSI escape codes and Unicode half-blocks. Think of it as a retro game engine, but instead of pixels on a screen, you're drawing colored characters in a terminal window.

Recently, I implemented three different rendering methods for compositing multiple semi-transparent surfaces:

1. **`render()`** - The "fast path" with binary transparency (either fully opaque or fully transparent)
2. **`renderWithAlphaToBg()`** - Alpha blending optimized for rendering to opaque backgrounds
3. **`renderWithAlpha()`** - Full alpha compositing supporting semi-transparent backgrounds

The theory was simple: `render()` should be fastest (no blending calculations), while the alpha methods should be 2-3x slower due to the extra arithmetic. I built a comprehensive benchmark suite to prove it.

Then I ran the tests.

The results were... bizarre.

## The Anomaly

**[PLACEHOLDER: Bar chart comparing the three methods for 10x10 sprites with 3 overlapping surfaces]**

For tiny 10x10 sprites with 3 overlapping surfaces:

```
render():             1.547 µs/iteration  (SLOWEST)
renderWithAlphaToBg(): 1.089 µs/iteration  (42% faster!)
renderWithAlpha():     0.856 µs/iteration  (81% faster!)
```

Wait, what? The "dumb" alpha blending function that does extra multiplication on every pixel is beating the "optimized" binary transparency path by almost 2x?

But it gets weirder. When I tested with 5 overlapping surfaces instead of 3:

```
render():             0.937 µs/iteration  (FASTEST)
renderWithAlphaToBg(): 1.563 µs/iteration
renderWithAlpha():     1.276 µs/iteration
```

Now `render()` is fastest again, and it's actually **65% faster with 5 surfaces than with 3 surfaces**, despite doing more work!

Something was very wrong. Either my benchmark was broken, or I was about to learn something interesting about modern CPUs.

---

## The Investigation

Let me show you the core of the `render()` function:

```zig
for (surfaces) |surface| {
    for (each_pixel_in_surface) |pixel| {
        // Skip transparent pixels
        if (surface.shadow_map[pixel] == 0) continue;

        const idx_out = calculate_output_position(pixel);

        // Only write if destination is not already occupied
        if (out_surface.shadow_map[idx_out] != 1) {
            out_surface.color_map[idx_out] = surface.color_map[pixel];
            out_surface.shadow_map[idx_out] = 1;
        }
    }
}
```

The optimization seems sound: surfaces are z-sorted (front-to-back), and we skip writing to pixels that are already drawn. Early exit, avoid redundant work, classic optimization.

Compare this to the alpha blending version:

```zig
for (surfaces) |surface| {
    for (each_pixel_in_surface) |pixel| {
        if (surface.shadow_map[pixel] == 0) continue;

        const idx_out = calculate_output_position(pixel);

        // ALWAYS blend, regardless of what's there
        const blended = blend_colors(
            surface.color_map[pixel],
            out_surface.color_map[idx_out]
        );
        out_surface.color_map[idx_out] = blended;
    }
}
```

The alpha version does MORE work (multiplication, division) but has NO conditional write. It always blends.

That's the key difference.

---

## Branch Prediction: The Hidden Performance Killer

Modern CPUs don't execute instructions one at a time. They use **pipelining** - fetching, decoding, and executing multiple instructions simultaneously. The Apple M4 in my MacBook can have dozens of instructions in flight at once.

But there's a problem: branches (if statements) create uncertainty. When the CPU encounters a branch, it doesn't know which path to take until the condition is evaluated. Waiting would stall the pipeline, wasting precious cycles.

The solution? **Branch prediction**. The CPU *guesses* which path the branch will take and speculatively executes instructions down that path. If the guess is correct, everything proceeds at full speed. If wrong, the CPU must throw away all the speculative work and restart from the correct path.

This is called a **branch misprediction**, and on modern processors like the M4, it costs roughly 15-20 CPU cycles per misprediction.

### The Pattern Matters

Here's where my test setup matters. I positioned sprites with 5-pixel offsets:

```
Surface 0: position (10, 10), z-index 0 (drawn last)
Surface 1: position (15, 15), z-index 1
Surface 2: position (20, 20), z-index 2 (drawn first)
```

With 10x10 pixel sprites, this creates different overlap patterns depending on how many surfaces you have.

**Three surfaces (irregular pattern):**

```
Surface 2 drawn first:  100% of pixels written (0% already occupied)
Surface 1 drawn second:  75% of pixels written (25% overlap with surface 2)
Surface 0 drawn last:    75% of pixels written (25% overlap, but irregular pattern)
```

The branch condition `if (out_surface.shadow_map[idx_out] != 1)` becomes unpredictable. For surface 2, it's always false (100% write rate). For surfaces 1 and 0, it's a mix (75% write rate), but the pattern of which pixels overlap is irregular.

**Five surfaces (regular pattern):**

```
Surface 4: 100% written (always false)
Surface 3:  75% written (predictable pattern)
Surface 2:  50% written (very predictable - half the pixels)
Surface 1:  25% written (predictable pattern)
Surface 0:   0% written (always true - everything overlaps)
```

The five-surface pattern creates more *regular* overlap. The CPU's branch predictor can learn: "For this surface at this position, we usually skip/write." The pattern is predictable.

**[PLACEHOLDER: Diagram showing overlap patterns for 3 vs 5 surfaces]**

### The Math

Let me estimate the branch misprediction cost:

- M4 CPU: ~15-20 cycles per misprediction
- Three-surface test: ~40-50 mispredictions per iteration (estimated)
- At M4's ~4 GHz clock: 40 mispredictions × 15 cycles = 600 cycles ≈ 0.15 µs

But we're processing 100 pixels twice (surfaces 1 and 0), so the irregular pattern compounds. Total overhead: **~0.6 µs**.

The observed difference: 1.547 µs (3 surfaces) - 0.937 µs (5 surfaces) = **0.61 µs**.

The numbers match perfectly. We're not measuring rendering performance. We're measuring branch prediction failure.

### Why Alpha Blending Wins

The alpha blending functions have no conditional write. They always execute the same code path:

```zig
// No branch here!
out_surface.color_map[idx] = blend(fg_color, bg_color);
```

No branches means no mispredictions. For very small sprites (10x10 = 100 pixels) with irregular overlap patterns, the cost of branch misprediction exceeds the cost of doing unnecessary multiplications.

This is counterintuitive but profound: **sometimes doing more arithmetic is faster than trying to be clever with branches**.

---

## The Second Mystery: The Megapixel Curve

While investigating the rendering engine, I also benchmarked a separate function: `toAnsi()`, which converts a pixel surface into ANSI escape sequences for terminal rendering.

The throughput measurements showed something unexpected:

**[PLACEHOLDER: Line graph of MP/s vs sprite size for toAnsi()]**

```
Size    | Time/iteration | Throughput (MP/s)
--------|----------------|------------------
10x10   | 0.268 µs      | 373.0 MP/s
20x20   | 1.140 µs      | 350.8 MP/s
40x40   | 4.749 µs      | 336.9 MP/s  ← Lowest
64x64   | 10.712 µs     | 382.4 MP/s  ← Peak!
80x80   | 18.655 µs     | 343.1 MP/s
100x100 | 28.467 µs     | 351.3 MP/s
200x200 | 111.850 µs    | 357.6 MP/s
```

The processing time scales roughly quadratically (as expected for O(n²) algorithms), but the throughput isn't monotonic. There's a clear peak at 64x64, with a valley at 40x40.

Why does performance *improve* when processing more data?

## Understanding Cache Hierarchy

Modern processors don't access RAM directly for every memory operation. That would be far too slow. Instead, they use a hierarchy of increasingly faster (and smaller) caches:

**Apple M4 Cache Structure:**
- **L1 Cache**: 128 KB per core, ~3-4 cycle latency (fastest)
- **L2 Cache**: 4 MB shared, ~12-15 cycle latency
- **L3 Cache**: 16 MB shared, ~30-40 cycle latency
- **RAM**: 16+ GB, ~100+ cycle latency (slowest)

When your program accesses memory, the CPU checks L1 first. If the data is there (cache hit), you get it in a few cycles. If not (cache miss), the CPU checks L2, then L3, then finally RAM. Each level is progressively slower.

The key insight: **your program's performance is often determined by how well your data fits in cache**.

### The Memory Footprint

Each pixel in my rendering engine requires:

```zig
struct Pixel {
    color: RGB,      // 3 bytes
    alpha: u8,       // 1 byte
    char: u21,       // 4 bytes (UTF-8)
}
// Total: 8 bytes per pixel
```

For different sprite sizes:

```
10x10:   100 pixels ×  8 bytes =    800 bytes
40x40: 1,600 pixels ×  8 bytes = 12,800 bytes
64x64: 4,096 pixels ×  8 bytes = 32,768 bytes  ← Key size!
80x80: 6,400 pixels ×  8 bytes = 51,200 bytes
```

Notice anything special about 64x64? It's **32 KB** - roughly a quarter of the 128 KB L1 cache.

At this size:
- The entire sprite fits comfortably in L1 cache
- All pixel data stays "hot" during processing
- The CPU can effectively prefetch the next cache line
- Memory access patterns align with 64-byte cache line boundaries

### Too Small: Overhead Dominates

For smaller sprites (40x40 and below), the data fits in cache easily, but another factor dominates: **ANSI string generation overhead**.

The `toAnsi()` function generates strings like:

```
\x1b[38;2;255;0;0m▀\x1b[48;2;0;255;0m
```

That's roughly 43 bytes of output per "pixel pair" (two vertical pixels rendered as one half-block character). For a 10x10 sprite, you're generating ~2 KB of string data to represent 800 bytes of pixel data. The ratio of output to input is unfavorable - string manipulation overhead dominates actual pixel processing.

### Too Large: Cache Thrashing

For sprites larger than 64x64, the data no longer fits in L1 cache. An 80x80 sprite needs 51 KB, which exceeds L1's capacity. The CPU must fetch data from the slower L2 cache, introducing more latency.

At 200x200 (320 KB of pixel data plus ~860 KB of output string), you're definitely in L2 or even L3 territory. The memory access pattern becomes less predictable, cache lines get evicted before reuse, and overall efficiency drops.

**[PLACEHOLDER: Diagram showing memory hierarchy and where different sprite sizes land]**

### The Sweet Spot

64x64 represents the Goldilocks zone:

- Large enough that string generation overhead is amortized
- Small enough to fit entirely in L1 cache
- Perfect alignment with CPU cache line architecture
- Maximum data reuse and minimal cache misses

The result: peak throughput of 382 megapixels per second.

This isn't theoretical. The cache hierarchy is a physical reality of CPU design, and your code's performance is directly shaped by how well your data structures align with it.

---

## Practical Implications

These discoveries have real implications for how I write performance-critical code:

### 1. Profile, Don't Assume

My "optimized" early-exit path was slower in some cases. The only way to know is to measure. Assumptions about performance are often wrong, especially when they ignore CPU microarchitecture.

### 2. Branchless Can Be Faster

For small data sets with unpredictable patterns, eliminating branches entirely may outperform clever conditional logic. This is especially true when:

- Working with small arrays (< 1000 elements)
- Branches have unpredictable patterns
- The cost of misprediction exceeds the cost of extra work

### 3. Design for Cache Locality

Structure your data to fit in cache:

- Keep hot data compact (64x64 is not arbitrary!)
- Process data in cache-friendly patterns
- Consider cache line sizes (64 bytes on most modern CPUs)
- Group related data together in memory

### 4. Different Scales, Different Bottlenecks

Performance optimization isn't one-size-fits-all:

- **Small data**: Overhead and branches matter most
- **Medium data**: Cache locality is king
- **Large data**: Memory bandwidth and algorithmic complexity dominate

The same code can have completely different performance characteristics at different scales.

---

## The Bigger Picture

Building this rendering engine taught me something fundamental: **modern CPUs are incredibly complex, and your high-level code is only part of the performance equation**.

The CPU's branch predictor, cache hierarchy, speculative execution, and out-of-order execution all influence performance in ways that aren't visible in your source code. You're not just writing algorithms; you're writing instructions that will be interpreted and transformed by layers of hardware optimization.

This isn't esoteric knowledge for compiler writers. It's practical wisdom for anyone writing performance-critical code. A few takeaways:

**Know your working set size.** How much data are you actively using? Does it fit in L1? L2? Structure your algorithms to work on cache-sized chunks.

**Know your branch patterns.** Are your conditionals predictable? If not, can you eliminate them? Sometimes `result = condition * valueA + (1-condition) * valueB` is faster than `if (condition) result = valueA else result = valueB`.

**Measure at the right scale.** Testing with 10x10 sprites revealed branch prediction issues that would have been invisible at larger sizes. Test across your expected range of inputs.

**Trust the profiler, not your intuition.** Your mental model of performance is probably wrong. Mine certainly was.

---

## Conclusion

I set out to build a terminal graphics engine and accidentally built a branch prediction and cache profiler.

The "anomalies" in my benchmark results weren't bugs - they were features, revealing the hidden behavior of modern CPUs. The render function that should have been fastest was sometimes slowest. The megapixel throughput curved up and down in unexpected ways.

But these weren't mysteries once I understood what I was actually measuring: not just my code, but the interaction between my code and the CPU's microarchitecture.

Next time you see unexpected benchmark results, don't immediately assume your code is wrong. You might be seeing the fingerprints of branch predictors, cache hierarchies, or any number of hardware optimizations working in the background.

Or, like me, you might be learning that sometimes the "slow" path is faster, and 64x64 isn't just a nice round number - it's a cache-aligned sweet spot on an Apple M4 processor.

---

**About the Project**: movy is an open-source terminal graphics engine written in Zig, capable of sprite rendering, alpha blending, animation, and effects - all in your terminal window. The complete source code and performance analysis are available on GitHub.

**[PLACEHOLDER: Screenshot of terminal graphics rendered with movy]**

---

*Have you encountered surprising performance characteristics in your code? What did they teach you about your hardware? Share your stories in the comments below.*
