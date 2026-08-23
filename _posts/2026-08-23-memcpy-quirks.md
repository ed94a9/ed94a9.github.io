---
title: std::memcpy is Not The Fastest Function For it's Purpose
date: 2026-08-23 18:21:00 +0800
categories: [Systems, C++]
tags: [c++, linux]
---

`std::memcpy` is a highly optimized function in libc implementations as it is one of the most fundamental operations on a chunk of computer memory. If you open up the glibc's implementation of it. You can see lots of hand-written intrinsics/assemblies that takes advantages of modern computer data-level parallelism to speed up the operations. Although there are other implementations out there claiming to be faster than glibc [rte_memcpy](https://www.intel.com/content/www/us/en/developer/articles/technical/performance-optimization-of-memcpy-in-dpdk.html), it is not the topic of this blog. In this blog, I am going to talk about some nuances about the speed/performance aspect of calling `std::memcpy`, especially when the payload is actually small.

Let me put the conclusion upfront here: `std::memcpy` is not as fast as you think when it comes to small buffer copy. You probably want to avoid it when it's in latency-critical cases.

## The Problem

I have been working in the quantitative finance industry, where system latency is sometimes the key that can decide whether you lose money or win money. My work have been continuously finding opportunities to accelerate the trading pipeline. The other day my attention was drawn to a very specific part of the system: The ticker. A ticker is just a string of a tradable instrument in the market and the literal characters are embedded in the market feeds. It's a semi-regular string in the sense that we know:

  1. The tickers cannot be outside of the full universe that day on the market.
  2. The tickers will not be very long: smaller than 16 characters, and most of the time, smaller than 8 characters.
  3. It's all in printable characters.

On the arrival of a piece of market data. The system will firstly assign a number (id) to the ticker so we don't have to use the string representation of the ticker everywhere in the system as that will be very slow in some cases like when you are maintaining a hashmap using the ticker string as the key. This handle has always been in the system. But it still cannot avoid 1 hashmap-like look up for tickers in the string form at the time when we try to assign the id to it -- after all, we need to know if the ticker in string has already been seen by the system.

After some profiling, it seems like that a naive look up using some best hashmap implementation takes on the order of **50 ns**, which does not sound quite a log, but it is a lot in a high frequency trading system. So here comes the question: Is there a way to optimize the has function of a ticker in string to accelerate the process. After all, it seems like 50 nanoseconds is way over the limit of this thing.


## The Perfect Hash

Perfect hash is the first thing that comes to my mind. It is a technique that aims to produce a collision-free hash function for a given set of elements upfront, for different kinds of reasons, accelerating hashmap look up being one of them. But this won't fit in our system because the full set of the available tickers on the market varies from day to day. We simply cannot afford to generate a perfect hash everyday and re-compile or re-link it to the trading binary. It's too much an operation burden and a very dangerous one -- If some day you recompile pipeline breaks or you forget to do it. You'll be facing with undefined behavior for certain. At that time, a crash is the best you can hope for. And you will be left with little clue how the system went off its track.


## The memcpy

Ok, well, I see, perfect has is not an option. So maybe a `memcpy` of the ticker to a 64-bit unsigned integer would be great idea ? After all, you can't go faster than `memcpy` right ?

