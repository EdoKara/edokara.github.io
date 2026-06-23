---
title: "Optimizing Geospatial Hashing in Rust"
layout: post
date: 2026-05-30
mermaid: true
---

## What Led me Here

My radar processing algorithm is *slow*. Across two phases, it takes
four minutes for one file. This is OK for bench testing
(bench in the lab sense, like bench-scale),
but if I want to actually run it in real time, it needs
some gas. One fringe benefit of piloting this procedure in
a jupyter notebook before scripting is that I have some basic
sense of what steps are the bottleneck in the process.

### The Main Bottlenecks

The major bottlenecks in this project are:

1. For the first part: mapping a layer from a weather model onto the radar quadmesh.
2. Interpolating the final result from radar coordinates
to a fixed output grid at the end.

This is a discussion about the first one; the second one will
need a lot more work to get right and I'm a lot more receptive
to doing something fast that's already developed.

## Scope of Work

This post covers just the *first* part of the algorithm that I
ultimately want to implement, because I went down
a rabbit hole trying to make this as fast as possible.
I'm not sure I actually succeeded, but I think what I
came up with is reasonably fast and it let me dip my
toes into some lower-level implementation work.

So the things I'll cover today are:

1. [The problem statement](#the-problem-statement)
2. [The indexing algorithm](#the-algorithm-in-words)
3. [How I implemented it in Rust](#how-i-implemented-it-in-rust)
4. [Benchmark results](#results-summarized)

## The Problem Statement

So, here's the problem statement for this part.

The main bottleneck for the first phase of my algorithm is
mapping a weather model's temperature output
(actually, the height of the freezing layer)
to each radar layer. My current implementation
requires building and traversing a tree over a subset
of the weather model's output (specificially the HRRR model)
and using radar points as query points. This is all done
with `Scipy`'s `BallTree` implementation. I'm sure the
`BallTree` is great, and it is certainly convenient to use their
model, but it is also a lot more general than I need it to be
in order to handle what I'm asking for, and performance
suffers because of that. My tree array is 250,000 observations
when subset, and the raw point query is ≈720x1860 (1,360,000) observations.
The largest test set shown in the docs for the excellent [herbie](https://herbie.readthedocs.io/en/stable/)
package are about 2 orders of magnitude smaller than that.

### Baseline Hunches & Prior Art

Based on what I've got here, I want to do as little work as possible to
compute a final answer. Some things helping me in this are:

1. The output grid for HRRR is fixed; I can cache it and query off of it.
2. It's possible to parallelize the grid querying by choosing a different
representation.

The first is reasonable, and in fact is the default behavior for the
basic implementation. The second isn't so obvious. I had a hunch that
it was possible, which some reading helped me confirm. The main
guide I'm following is an excellent series of articles from the
NVIDIA technical blog on tree traversal:

- [Parallel Tree Traversal: Part 1](https://developer.nvidia.com/blog/thinking-parallel-part-i-collision-detection-gpu/)
- [Parallel Tree Traversal: Part 2](https://developer.nvidia.com/blog/thinking-parallel-part-ii-tree-traversal-gpu/)
- [Parallel Tree Traversal: Part 3](https://developer.nvidia.com/blog/thinking-parallel-part-iii-tree-construction-gpu/)

Part 3 in particular is the part that I'm following as an implementation guide,
but I'll explain in words the idea of what I'm doing to pull out the nugget of the
idea.

## The Algorithm in Words

Here's what the algorithm looks like, using words:

1. Take the latitude and longitude coordinates
and transform them to unit reference (i.e. (0,1])
2. Turn the lat/lon `f64`s into fixed-point numbers on this interval.
3. pull out that fixed precision as an `int32`
4. interleave the latitude and longitude bits from the `int32`
to make a [Morton Code](https://en.wikipedia.org/wiki/Z-order_curve).

This is what we're covering here; the next step is actually
performing efficient indexing and search on what is effectively
an compact encoding of a tree.

The basic logic behind implementing the logic this way is that
we can match, for the same precision, both sets to each other
in terms of a spatial bucket of a given size. In this case, the
resolution of the model dataset is coarser so it lets us
construct the smallest representation of the problem overall.

The Z-order curve we're trying to construct here is
useful from an efficiency perspective because we don't have
to compute anything especially hard to end up with it,
and we don't have to build a big, complex data structure
to access the hierarchical organization of the dataset,
just encode it a different way. That lets us also operate
on all items independently of each other, unlocking
parallelization.

The main thing that we have to handle is some bit swizzling and
integer multiplication, which is not something I'm super
familiar with and that requires lots of pictures and head
scratching to get right, along with judicious referral
to
[excellent blog posts](https://lemire.me/blog/2018/01/09/how-fast-can-you-bit-interleave-32-bit-integers-simd-edition/)
on the matter.

## How I implemented It in Rust

First let's talk about the basic reference implementation,
which is scalar. This one is *kind of* the simplest to write,
but the bit interleaving still requires some interpretation and
squinting, which I'll talk you through (for my benefit since explaining it is
helping me learn).

The basic steps of scaling the input and converting it to fixed are:

```rust
#[inline(always)]
fn scale_lat(n: f64) -> f64 {
  n / 90.
}
```

Note that this constrains my problem to the northern hemisphere,
and in fact the E-W direction also constrains my solution to be
north of the equator. Lucky for me, here at 43°, myself and
my dataset subset are very comfortably within that area. I'm
doing inline(always) because this is about the most ridiculous
function to *not* inline, and I want to be clear with the compiler
that it's fully OK to optimize this away in-place where it's found.

Converting to fixed point after this is basically a matter of
a bit shift:

```rust
#[inline(always)]
fn to_fixed(n: f64) -> i32 {
  (n * ((1 << 20) as f64)) as i32
}
```

The effect of this function is to move the bits
of the float $2^{20}$ over and truncate the mantissa, leaving only the
set of bits of specified precision from the float.

These are the easy parts, and in practice they get rolled into one
scale/fix function. Then we get to the more
fun parts of the procedure. What helped me understand this
was, again, the
[excellent blog post](https://lemire.me/blog/2018/01/09/how-fast-can-you-bit-interleave-32-bit-integers-simd-edition/)
on bit interleaving. For those unaware (like myself before starting this)
the basic operation looks like this:

```rust
    word = (word ^ (word << 16)) & 0x0000ffff0000ffff;
```

Splitting this first step up gives you:

1. bitwise XOR (`^`) the word (cast to `i64`) with
itself shifted 16 bits to the right.
2. bitwise AND this product with a mask to only carry
some of the bits forward.

Recall that we started out with an `i32`. This means that, when
we run `<<16`, we have a 16-bit overlap between the two copies.
The xor means that anything that matches will carries forward
the 1s that don't match up on the overlap. The `&` operation
then masks out the middle values, leaving us with the high
and low bit halves split by 16 bits. I wrote out an example
to prove it to myself. These go ORIGINAL -> OPERAND -> RESULT:

```text
XOR
00000000000000000000000000000001110110101110011001111001100110
00000000000000011101101011100110011110011001100000000000000000
00000000000000011101101011100111101000110111111001111001100110

AND
00000000000000011101101011100111101000110111111001111001100110
00000000000000011111111111111110000000000000000111111111111111
00000000000000011101101011100110000000000000000001111001100110

Comparison:
00000000000000011101101011100110000000000000000001111001100110
00000000000000000000000000000001110110101110011001111001100110
```

This process then gets repeated at smaller steps to introduce
smaller splits into the overall product. The whole operation
looks like:

```rust
    word = (word ^ (word << 16)) & 0x0000ffff0000ffff;
    word = (word ^ (word << 8)) & 0x00ff00ff00ff00ff;
    // bit shifts 4 over & makes two copies, grabbing
    // only the lower one again in a smaller pattern.
    // 0f is 00001111
    word = (word ^ (word << 4)) & 0x0f0f0f0f0f0f0f0f;
    // 3 =  0011;
    word = (word ^ (word << 2)) & 0x3333333333333333;
    // 5 = 0101;
    word = (word ^ (word << 1)) & 0x5555555555555555;
```

in this case, our progressively smaller masks are:
 `00ff` = `00001111`, `0f` = `0011`, `3` = `0011`, and `5` is `0101`.

So it makes sense that we're just repeating the
same operation, more or less, for 5 steps that
interleave zeros at more granular levels between these,
introducing enough space at each step to accommodate
the spreading out that's happening.

This operation is applied to the `latitude` and `longitude`
bits separately, and then they are inteleaved by
simply doing:

```rust
(lon_bits << 1) | lat_bits
```

which is a normal bitwise OR (so if either bit is 1, it carries forward)
while shifting one over so that there's not register collision between
the two values. This ends up giving us a code combining
latitude and longitude for a point and *should* uniquely encode
the point to the precision we've chosen.

### Optimizing

I decided to optimize this part before going
any further with the rest of the algorithm, and I did so for two reasons.
Primarily, because it's fun (for my sick, twisted definition),
and secondarily, because it also makes sense to. My plan is to
deploy this for a processing pipeline that's going to run on *at least* 374 million
coordinates per day (not counting the array I'm indexing into).

So, how did I optimize it?

Well, I'm not super across my bitwise algorithms, so instead of
going into the math mines like I'm [G optimizing the Fibonacci sequence](https://www.youtube.com/@SheafificationOfG),
I took a much more straightforward approach and wrote some SIMD. For those
unaware and still reading (go you) SIMD is *Single Instruction, Multiple Data*.
Instead of loading one value into a register at a time, a SIMD register loads
a large number together, with each number residing in a lane; all of these
lanes can be operated over in parallel, which can substantially improve the
throughput of numeric operations.

My CPU is an AMD Ryzen 9 7950x clocked at 5.88 GHz. `lscpu` helpfully
informs me that I have a number of AVX512 extensions that I can take advantage of.
So that sets out the main strategy; simply do the operations in parallel. Each
of my cores *should* (I couldn't figure out exactly) have 32 512-bit registers,
which means I can theoretically have 32kB of data in registers at the same time across
all my cores.

My baseline benchmark for the scalar version is:

```
converts                                  fastest       │ slowest       │ median        │ mean          │ samples │ iters
╰─ zcode_latlon_scalar_many_singlethread  5.58 ms       │ 6.311 ms      │ 5.655 ms      │ 5.675 ms      │ 100     │ 100
```

Note that the names will continue to be at least that awful, since
I tried like 20 variations of this before settling on this one. So
it takes about 5ms for this baseline bench to handle 100 lat/lon
entries.

Converting to SIMD involved squinting at a bunch of intrinsic names,
following some basic patterns, and dealing with some `unsafe`. In
this case, we're swimming in relatively friendly waters where `unsafe`
is concerned, so that part at least I'm not particularly worried about.
Our original functional pattern for hashing under simd becomes:

```rust
 pub fn simd_zero_interleave(mut word: __m512i) -> __m512i {
    // safety: ... (ellided for compactness)
     unsafe {
         
         let m1: __m512i = _mm512_set1_epi64(0x0000ffff0000ffff);
         let m2: __m512i = _mm512_set1_epi64(0x00ff00ff00ff00ff);
         let m3: __m512i = _mm512_set1_epi64(0x0f0f0f0f0f0f0f0f);
         let m4: __m512i = _mm512_set1_epi64(0x3333333333333333);
         let m5: __m512i = _mm512_set1_epi64(0x5555555555555555);
 
         word = _mm512_xor_si512(word, _mm512_slli_epi64(word, 16));
         word = _mm512_and_si512(word, m1);
         word = _mm512_xor_si512(word, _mm512_slli_epi64(word, 8));
         word = _mm512_and_si512(word, m2);
         word = _mm512_xor_si512(word, _mm512_slli_epi64(word, 4));
         word = _mm512_and_si512(word, m3);
         word = _mm512_xor_si512(word, _mm512_slli_epi64(word, 2));
         word = _mm512_and_si512(word, m4);
         word = _mm512_xor_si512(word, _mm512_slli_epi64(word, 1));
         word = _mm512_and_si512(word, m5);
         word
     }
 }
```

A couple of things about this code, breaking it down:
1.instead of the nice scalar functions we can normally work with,
we end up working with a specific intrinsic. These intrinsics
make the code extremely ugly.
2.The format is *almost* the same as before. We're taking `__m512i` registers as
inputs instead of an `i64`.

Other than that, it's the same. Briefly, `__m512i` registers are a bit squishy,
in that they can be reinterpreted in-place as 8,16,32, or 64-bit integers, so
it's important to be careful about what you're treating things as when. One nice
thing is that the intrinsics are patterned to sort of signpost for maximum specificity.
In this case:

```text
__mm512_ | _slli_   | epi64 |
512-bit  | slide    | as i64|
register | left(int)|       |
```

As an example. If I wanted the 256-bit variant, I just swap `__mm512_` for `__mm256_`
and carry on as before.

Loading values into the registers is comparatively expensive as well, so while I'm
doing my bit shifts in them I can also scale everything and keep stuff in the
registers for multiple steps, which should save some time if it actually
materializes.

When we pull all this together with iterators,
it looks something like this:

```rust
pub fn interleave_latlon_allsimd(lat: Vec<f64>, lon: Vec<f64>) -> Vec<i64> {
      let sf_lat = lat.chunks_exact(8).map(|chunk| unsafe {
          let word = _mm512_loadu_pd(chunk.as_ptr());
          let fixed_repr = to_fixed_simd(scale_lat_simd(word));
          simd_zero_interleave(fixed_repr)
      });
  
      let sf_lon = lon.chunks_exact(8).map() //... the same as above
  
      let mut out_vec: Vec<i64> = sf_lat
          .zip(sf_lon)
          .map(|(lat, lon)| unsafe {
              let jndreg: __m512i = _mm512_or_si512(_mm512_slli_epi16(lon, 1), lat);
              let mut outbuf = [0i64; 8];
              _mm512_storeu_epi64(outbuf.as_mut_ptr(), jndreg);
              outbuf
          })
          .flatten()
          .collect();
  
      let lonrem = lon.chunks_exact(8).remainder();
      let latrem = lat.chunks_exact(8).remainder();
  
      // this part is to collect the tail at the end of the
      // function if there's anything left.
      let tail_zcodes = lonrem
          .into_iter()
          .zip(latrem.into_iter())
          .map(|(lon, lat)| naive_zcode_latlon(*lat, *lon));
  
      out_vec.extend(tail_zcodes);
      out_vec
  }
```

This iteration strategy takes lazy iteration chunks of both
latitude and longitude and runs the zero interleave independently
on each, then pulls them together and spits the output into
an allocated buffer that we immediately flatten. The way
I imagine this working in my idealized situation is that
as each OR operation happens, a pipeline of zero offset operations
are occurring behind it such that everything runs smoothly together.
I'm pretty sure that's not 100% what's happening, but I do know
that we get a speedup from this method.

```text
                                      fastest       │ slowest       │ median        │ mean          │ samples │ iters
╰─ zcode_latlon_allsimd_many_sthread                │               │               │               │         │
   ├─ 5                               61.44 µs      │ 84.87 µs      │ 61.98 µs      │ 62.61 µs      │ 100     │ 100
   ├─ 10                              251.7 µs      │ 270.7 µs      │ 257.2 µs      │ 257.9 µs      │ 100     │ 100
   ├─ 50                              5.442 ms      │ 6.218 ms      │ 5.489 ms      │ 5.516 ms      │ 100     │ 100
   ├─ 100                             22.09 ms      │ 24.22 ms      │ 22.27 ms      │ 22.32 ms      │ 100     │ 100
   ├─ 200                             88.22 ms      │ 95.28 ms      │ 88.91 ms      │ 89.07 ms      │ 100     │ 100
   ├─ 300                             209.6 ms      │ 213.5 ms      │ 211.2 ms      │ 211.2 ms      │ 100     │ 100
   ├─ 500                             585.6 ms      │ 598.8 ms      │ 593 ms        │ 592.8 ms      │ 100     │ 100
   ╰─ 1000                            2.317 s       │ 2.36 s        │ 2.326 s       │ 2.329 s       │ 100     │ 100

```

Each of these inputs is generated similarly to an `np.meshgrid`,
so the '500' input ends up making the equivalent of two 500x500 arrays (500,000 points).
Compared to our first version, this one is substantially faster.
Our first entry could handle 100 lat/lons in 5ms,
and this version can handle $$ 50^{2} \times 2 = 500$$ points, so we're
going about 5 times faster. Realistically though, we're getting most of our efficiency gains
from larger input sizes at this point. We're also scaling roughly linearly, so
we're doing about 100 times the work in 100x the time.

We can take this further by unrolling our loop. This requires a little restructuring of our
logic from the previous iteration approach. I'll give you a snippet of the function for flavor.

```rust
pub fn interleave_latlon_allsimd_unroll(lat: Vec<f64>, lon: Vec<f64>) -> Vec<i64> {
    let sf_lat = lat.chunks_exact(8 * 32).map(|chunk| unsafe {
        let word1 = _mm512_loadu_pd(chunk[0..8].as_ptr());
        let word2 = _mm512_loadu_pd(chunk[8..16].as_ptr());
        let word3 = _mm512_loadu_pd(chunk[16..24].as_ptr());
        let word4 = _mm512_loadu_pd(chunk[24..32].as_ptr());
// ... it goes on like this for a long time
```

I was really thankful for vim macros while writing this function. You can see an important
difference between this and the prior version in the start of the iterator. I'm working over
chunks of $$8\times 32 = 256$$ now, which in theory should completely fill the registers
when we're done. If I was able to stick to `i32` exclusively this could double, but since we
go out to `i64` I can't fit more in since we have to expand each number. The benchmark is
promising, there is a consistent 15%ish speedup in the larger array benchmarks here from
unrolling the loop.

```text
                                    fastest       │ slowest       │ median        │ mean          │ samples │ iters
╰─ latlon_allsimd_unrolled_sthread                │               │               │               │         │
   ├─ 5                             61.62 µs      │ 83.18 µs      │ 62.12 µs      │ 62.61 µs      │ 100     │ 100
   ├─ 10                            220.4 µs      │ 265.8 µs      │ 227.6 µs      │ 229.2 µs      │ 100     │ 100
   ├─ 50                            4.839 ms      │ 5.199 ms      │ 4.881 ms      │ 4.887 ms      │ 100     │ 100
   ├─ 100                           19.36 ms      │ 20.17 ms      │ 19.77 ms      │ 19.73 ms      │ 100     │ 100
   ├─ 200                           81.55 ms      │ 93.4 ms       │ 89.91 ms      │ 89.67 ms      │ 100     │ 100
   ├─ 300                           180.2 ms      │ 183.4 ms      │ 181.1 ms      │ 181.2 ms      │ 100     │ 100
   ├─ 500                           500.4 ms      │ 505.9 ms      │ 502.9 ms      │ 502.9 ms      │ 100     │ 100
   ╰─ 1000                          1.982 s       │ 2.006 s       │ 2 s           │ 1.999 s       │ 100     │ 100
```

Once again, the differences are much more clear at larger intervals,
which makes perfect sense, since if this operates on less than a chunk
at any point, the function is defaulting to the scalar implementation to
clean up the tail. The less of our time we can spend on that operation,
the faster overall this operation will be. In fact, one optimization I neglected
(so far) was to run just rolled-up SIMD version on the remainder from the
unrolled chunk, and then only do the scalar tail operation on the last
7 or fewer of the items.

The last thing for us to do, now that we've gotten a reasonable upgrade
in our single-threaded performance, is to parallelize this guy over
all our cores. This is where [`rayon`](https://docs.rs/rayon/latest/rayon/) comes
in handy. This is just a matter of replacing our `iter`s with `par_iters`, and our
chunking with `par_chunks`. So the first line goes:

```rust
  let sf_lat = lat.par_chunks_exact(8 * 32).map(|chunk| unsafe {
  //... the whole unrolled mess
  } // the rest of the function as before...
```

This results in a substantial speedup at larger array sizes:

```text
                                  fastest       │ slowest       │ median        │ mean          │ samples │ iters
╰─ latlon_allsimd_unrolled_rayon                │               │               │               │         │
   ├─ 5                           272.3 µs      │ 505.5 µs      │ 328.4 µs      │ 350.1 µs      │ 10      │ 50
   ├─ 10                          432.5 µs      │ 708.7 µs      │ 501.5 µs      │ 523.2 µs      │ 10      │ 50
   ├─ 50                          3.317 ms      │ 3.764 ms      │ 3.36 ms       │ 3.415 ms      │ 10      │ 50
   ├─ 100                         10.57 ms      │ 13.62 ms      │ 10.88 ms      │ 11.18 ms      │ 10      │ 50
   ├─ 200                         31.76 ms      │ 40.16 ms      │ 33.01 ms      │ 33.6 ms       │ 10      │ 50
   ├─ 300                         69.24 ms      │ 77.65 ms      │ 72.27 ms      │ 72.77 ms      │ 10      │ 50
   ├─ 500                         179.9 ms      │ 193.8 ms      │ 183.6 ms      │ 184.7 ms      │ 10      │ 50
   ├─ 1000                        781.5 ms      │ 881.1 ms      │ 794.4 ms      │ 805.2 ms      │ 10      │ 50
   ├─ 1250                        1.355 s       │ 1.44 s        │ 1.398 s       │ 1.396 s       │ 10      │ 50
   ╰─ 2000                        4.556 s       │ 4.821 s       │ 4.755 s       │ 4.735 s       │ 10      │ 50
```

Now we can handle $$2 \times 1000^{2} = 2000000$$ element arrays in under a second.
Compared to the single-threaded version, this represents a 55% speedup, more
than halving the runtime. Processor occupancy looks OK when I'm running through
iterations of this benchmark, but I suspect I can do better somewhere, and that a
flamegraph would help me do so. Note also that we're paying for speed at larger
scales by going slower with smaller arrays.

![Processor occupancy for the multithreaded SIMD run]({{ site.baseurl }}/images/cpu_occupancy.png)

### Results Summarized

Pulling out the main results here to compare our progression:

| Function | Input Size | Median | Speedup |
| ---------|------------|--------|---------|
|simd_rolled_up |1000000 | 2.36s  | 0%      |
| simd_unrolled_sthread|1000000|2.00s|15%   |
|simd_unrolled_rayon|1000000|0.794s|66%|

#### Aside: Benchmarking

I did my benchmarks with the
[divan](https://github.com/nvzqz/divan) crate, which I'm pretty happy with
for this kind of experimenting. I especially like the
syntax for specifying multiple inputs for the same
function so you can observe scaling behavior.

## Next Steps

The primary next step is actually using this
hashing strategy to match up values across the two
arrays and then returning the result to python. This will
involve additional search and retrieval steps that
should hopefully have some interesting learning
along with them.
