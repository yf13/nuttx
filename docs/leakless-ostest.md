# Leakless ostest

This document findings on the kernel memory growth issue when running `ostest` app.

## The Problem

The NSH command `free` shows memory information like below:

```
ABC
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33378172       7036   33371136       7020   33371120     23      2
```

Where we can notice the used memory in kernel is `7036` bytes after a boot. After using some built-in commands or the simple `hello` app, it doesn't change.

However, after using the `getprime 4` app, the used memory grows:

```
nsh> free
                total       used       free    maxused    maxfree  nused  nfree
      Umem:  33377308       7052   33370256      18324   33363472     23      4
```

It grows further after running the `ostest` app:

```
nsh> free
                total       used       free    maxused    maxfree  nused  nfree
      Umem:  33377308       9140   33368168      46012   33363616     42      7
```

Are these implying that we are leaking memory?

## Findings

After some investigations, we have findings below:

- There is undelivered message leakage in timed mqueue that is fixed in [patch 12402](https://github.com/apache/nuttx/pull/12402). This is a true leak that is triggered by the `ostest` app.

- There is a global list of free sigaction objects in NuttX which are allocated dynamically and never freed after use. Thus when `ostest` app runs for the first time, used memory grows due to their allocation, but they don't grow if `ostest` app runs again. [Patch 12406](https://github.com/apache/nuttx/pull/12406) adds pre-allocated sigaction list and reclaims dynamically allocated ones timely thus we won't see the initial growth any more.

- There is a pid hash table which is a dynamic array of TCB pointers. The array has a very small initial length, thus when multi-threading apps like `getprime` or `ostest` are used, it grows quickly to accomdate more thread ids. The array never shrinks currently. The frequent initial growth feels like leakage. so [patch 12427](https://github.com/apache/nuttx/pull/12427) contains a solution to avoid frequent growth.

- There are a few folders created by `ostest` when doing mqueue tests, they are not cleaned timely thus their inodes are still using kernel memory.

## Result

After applying above patches, with proper configuration and clean up commands, we can see stable system memory usage after running `ostest` like below:

```
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33374748       7084   33367664       7068   33367632     21      2
nsh> ostest >/dev/null >>/dev/null
stdio_test: write fd=2
stdio_test: Standard I/O Check: fprintf to stderr
setvbuf_test: Using NO buffering
setvbuf_test: Using default FULL buffering
setvbuf_test: Using FULL buffering, buffer size 64
setvbuf_test: Using FULL buffering, pre-allocated buffer
setvbuf_test: Using LINE buffering, buffer size 64
setvbuf_test: Using FULL buffering, pre-allocated buffer
nsh> echo $?
0
nsh> rm -r /var
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33374748       7084   33367664      46108   33367632     21      2
```

So with a few improvements, our NuttX build can finish `ostest` cleanly.

Though real world app situations may vary, but for simple apps like `ostest`, being clean makes people more confident about NuttX.
