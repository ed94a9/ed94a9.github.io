---
title: `memcpy` quirks
date: 2026-08-23 18:21:00 +0800
categories: [Systems, C++]
tags: [c++, linux]
---

`std::memcpy` is wildly optimized function in libc implementations as it is one of the most fundamental operations on a chunk of computer memory. If you open up the glibc's implementation of it. You can see lots of hand-written intrinsics/assemblies that takes advantages of modern computer data-level parallelism to speed up the operations. Although there are other implementations out there claiming to be faster than glibc.
