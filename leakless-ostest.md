

## Problem

The NSH command `free` shows memory information like below:

```
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33378172       7036   33371136       7020   33371120     23      2
```

Where we can notice the `used` memory in kernel heap is 7036 bytes after a clean boot.

How this number changes during the use of NuttX? After using some built-in commands or the simple `hello` app, it doesn't change.

However, after using the `getprime 4` app, the used memory grows:

```
nsh> free
                   total       used       free    maxused    maxfree  nused  nfree
      Umem:     33377308       7052   33370256      18324   33363472     23      4
```

It grows further after running the `ostest` app once:

```
nsh> free
                   total       used       free    maxused    maxfree  nused  nfree
      Umem:     33377308       9140   33368168      46012   33363616     42      7
```

Are these implying that the kernel is leaking memory?

## Findings

After some investigations, we have a few findings:

- There is a pid hash table which is actually dynamic array of TCB pointers. The table has a very small initial length, thus when multi-threading apps like `getprime` or `ostest` are used, the table grows to accomdate more thread ids. The table doesn't shrink currently. This isn't a real leakage but it is a little misleading, and frequent growth may lead to more memory fragements. So patch 12427 adds a `PIDHASH_INITIAL_LENGTH` so that to avoid unnecessary growth.

- There is undelivered message leakage in timed mqueue source, as resolved in patch 12402. This is a true leak. This is triggered by `ostest` app usage.

- There is a global list of free sigaction objects which are allocated dynamically and never freed after use. Thus when `ostest` app runs for the first time, used memory grows due to their allocation, but they don't grow if `ostest` app runs again. Patch 12406 adds pre-allocated sigaction list and reclaims dynamically allocated ones timely thus we won't see used memory growth due to this list.

- There is a few folders created by `ostest` when using mqueue functions, they are not cleaned timely thus their inodes are using kernel memory.

## Result

After applying above patches and with proper clean up commands, we can see stable used memory before and after using the `ostest` app:

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
inode_alloc: 0x8002e6a8 var/mqueue/mqueue
inode_alloc: 0x8002e6d0 mqueue/mqueue
inode_alloc: 0x80030ff8 mqueue
nxmq_alloc_msgq: 0x80031028
inode_alloc: 0x80030ff8 timedmq
nxmq_alloc_msgq: 0x80031028
nxmq_alloc_msg: 0x80031090
nxmq_alloc_msg: 0x800310c0
nsh> echo $?
0
nsh> rm -r /var
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33374748       7084   33367664      46108   33367632     21      2
```

So with a few improvements, our build of NuttX not only finishes `ostest`, but also does it cleanly!

