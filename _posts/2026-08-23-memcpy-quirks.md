---
title: std::memcpy is Not Always The Fastest Function For it's Purpose
date: 2026-08-23 18:21:00 +0800
categories: [Systems, C++]
tags: [c++, linux]
---

`std::memcpy` is a highly optimized function in libc implementations as it is one of the most fundamental operations on a chunk of computer memory. If you open up the glibc's implementation of it. You can see lots of hand-written intrinsics/assemblies that takes advantages of modern computer data-level parallelism to speed up the operations. Although there are other implementations out there claiming to be faster than glibc [rte_memcpy](https://www.intel.com/content/www/us/en/developer/articles/technical/performance-optimization-of-memcpy-in-dpdk.html), it is not the topic of this blog. In this blog, I am going to talk about some nuances about the speed/performance aspect of calling `std::memcpy`, especially when the payload is actually small.

Let me put the conclusion upfront here: `std::memcpy` is not as fast as you think when it comes to small buffer copy. You probably want to avoid it when it's in latency-critical cases.

## The Problem

I have been working in the quantitative finance industry, where system latency is sometimes the key that can decide whether you lose money or win money. My work have been continuously finding opportunities to accelerate the trading pipeline. The other day my attention was drawn to a very specific part of the system: The ticker. A ticker is just a string of a tradable instrument in the market and the literal characters are embedded in the market feeds. It's a semi-regular string in the sense that we know:

  1. The tickers cannot be outside of the full universe that day on the market.
  2. The tickers will not be very long: up to 8 characters in length.
  3. It's all in printable characters.

On the arrival of a piece of market data. The system will firstly assign a number (id) to the ticker so we don't have to use the string representation of the ticker everywhere in the system as that will be very slow in some cases like when you are maintaining a hashmap using the ticker string as the key. This handle has always been in the system. The implementation was always relying on maintaining a string-to-integer hashmap. We thought that one hashmap was unavoidable and minimizing the number of string-keyed hashmaps to this essential one was the best we can do.

But after some profiling, we were still not happy with it -- one update on that hashmap along is measured to take ~**50 ns**, which does not sound quite a lot, but it is a lot in the world of high frequency trading.

Then there was an idea floating around in the team: Can we entirely remove that hashmap ? To put it another way, that is to come up with a scheme to assign each ticker an unique integer with the guarantee of: 1. No collision within the set; 2. No usage of hashmap.

Of cause we can!

## Pre-computed Perfect Hash

Perfect hash is the first thing that comes to my mind. It is a technique that aims to produce a collision-free hash function for a given set of elements upfront, for different kinds of reasons, accelerating hashmap look up being one of them since collision can lead to severer degradation of hash map performance. But this won't fit in our system because the full set of the available tickers on the market varies from day to day. We simply cannot afford to generate a perfect hash everyday and re-compile or re-link it to the trading binary. It's too much an operation burden and a very dangerous one -- If some day you recompile pipeline breaks or you forget to do it. You'll be facing with undefined behavior for certain. At that time, a crash is the best you can hope for. And you will be left with little clue how the system went off its track.


## `memcpy`

Ok, I see, perfect has is not an option. So maybe a `memcpy` of the ticker to a 64-bit unsigned integer would be great idea ? After all, you can't go faster than `memcpy` right ? A `mmecpy` will also guarantee the hash values will be collision-free.

Then I was immediately off to an experiment with the `memcpy`:

```cpp
std::unit64_t int_rep_1( const Ticker& ticker) // Ticker is an wrapper object of the actual char* representaion of the ticker.
                                               //    With the same interface of std::string
{
  std::uint64_t res{};
  std::memcpy( &res, ticker.data(), ticker.size() );
  return res; 			       // Should definitely have NRVO
}
```

After some benchmarking, this function takes 20+ ns on our machine, which is much better than 50 in the old system. Problem solved, isn't it ? You can out perform the memcpy by quite a lot can you ? But then I also took a measurement on the latency of `std::hash<std::string>::operator()` -- It was 5 ns, 1/4 of what `std::memcpy` gives.

That was a suprise to me. I've always thought `std::memcpy` is heavily optimized, and in this case, you have to touch all of that ticker's bytes in memory, then how can you be even faster than simply copying that memory blob into the return value register ?

## The Disassembly

After disassemblying the function. Things was a bit clearer: **The call to std::memcpy was not inlined**. That clears the mist: By using `std::memcpy`, we introduced an unwanted function call indirection in the code.

## The Modern Feature

After some digging. It appears that the compiler will resist to inline the `std::memcpy` function call for you when it has no idea how long the memory blob you are going to copy because it has no idea if it will be good or bad to the performance.

Then something caught my eyes: a new feature in C++20, the `assume` annotation, with the syntax in the form of `[[assume(/*The annotations you would like to add*/)]];`. I have not used this feature before. But this seems like a legit, genuine place for it. So then then I put up an annotation in the function:

```cpp
std::uint64_t int_rep_2( const Ticker& ticker) // Ticker is an wrapper object of the actual char* representaion of the ticker.
                                               //    With the same interface of std::string
{
  std::uint64_t res{};
  const std::size_t cpy_size = ticker.size();
  [[assume(cpy_size <= 8)]];                 // Added assumption
  [[assume(cpy_size >= 1)]];                 // Added assumption
  std::memcpy( &res, ticker.data(), cpy_size );
  return res;
}
```

Adding the two assumption lines makes the indirect call of `memcpy` disappear in the disassembly. Here is the proof: `https://godbolt.org/z/j8f48K7sP` . The new assumption annotation works!

Albeit, the result is still a little discouraging: The latency of the new function is not improving. The reason is also simple: Although the function call indirection disappears, too many codes are inlined, putting a lot of L cache pressure in the hot path, which is not necessary.

## The Simple Solution

So seems like the `std::mmecpy` is simply a too heavy tool in this scenario. So, many a very simple solution can save us ?

```cpp
std::uint64_t int_rep_3( const Ticker& ticker )
{
  switch ( ticker.size() ) {
    case 1:
      return static_cast<std::uint64_t>( ticker[0] );
    case 2:
      return (static_cast<std::uint64_t>( ticker[0] ) << 7) + static_case<std::uint64_t>( ticker[1] );
    case 3:
      return (static_cast<std::uint64_t>( ticker[0] ) << 14) + ( static_case<std::uint64_t>( ticker[1] << 7 ) + static_case<std::uint64_t>( ticker[1];
    // ... until 8
    default:
      throw std::runtime_error("Invalid ticker!");
  }
}
```

This very simple (almost) re-implementation of memcpy gives us something almost identical to the `std::hash` in terms of the latency. Which is very ideal. 

## Lessons

Some simple lessons:

- `std::memcpy` is not your best choice when you are dealing with small buffers. You will either face with function call indirections or high instruction cache pressure. There also seems to implementations claim to beat `std::memcpy` in general cases in speed, such as (DPDK)[[https://github.com/DPDK/dpdk]];
- `[[assume()]];` annotation works, which is almost like magic;


