# Linux Slab Allocator

## Basic Concepts

- **<u>Slab</u>**: A set of one or more **contiguous** pages of memory that contain kernel objects of a specific size.

- **<u>Slab Cache</u>**: Basically, it's a container of multiple slabs of the same type. There are two classes of caches:
  - **<u>Dedicated</u>**: They are created by the Linux kernel, and each one is used to hold only given type of a (commonly used) object such as `mm_struct` or `cred_jar`
  - **<u>Generic</u>**: General purpose caches that can hold any object of a specific size (plus padding). Usually, these caches have objects of sizes of power of two. When calling `kmalloc()` the, the returned allocated memory will be in one of such caches (like `kmalloc-32`, `kmalloc-64`, ...) depending on the size requested.

- **<u>SLOB, SLAB, SLUB</u>**: These are different slab allocators available in the Linux Kernel. Usually, nowadays the SLUB allocator is the default one. SLOB was the original allocator (first implemented in Solaris OS), while SLAB was an improvement upon it. SLUB came later as a simplification and improvement on SLAB.

To show information on the different slab caches we can execute:

```bash
$ sudo cat /proc/slabinfo
slabinfo - version: 2.1
# name            <active_objs> <num_objs> <objsize> <objperslab> <pagesperslab> : tunables <limit> <batchcount> <sharedfactor> : slabdata <active_slabs> <num_slabs> <sharedavail>                        
nf_conntrack         425    425    320   25    2 : tunables    0    0    0 : slabdata     17     17      0
ovl_inode             44     44    720   22    4 : tunables    0    0    0 : slabdata      2      2      0
kvm_async_pf           0      0    136   30    1 : tunables    0    0    0 : slabdata      0      0      0
kvm_vcpu               2      2  10944    1    4 : tunables    0    0    0 : slabdata      2      2      0
kvm_mmu_page_header    748    748    184   22    1 : tunables    0    0    0 : slabdata     34     34      0
x86_emulator          24     24   2672   12    8 : tunables    0    0    0 : slabdata      2      2      0  
...
kmalloc-8k           351    356   8192    4    8 : tunables    0    0    0 : slabdata     89     89      0
kmalloc-4k          2835   2992   4096    8    8 : tunables    0    0    0 : slabdata    374    374      0
kmalloc-2k          2513   2672   2048   16    8 : tunables    0    0    0 : slabdata    167    167      0
kmalloc-1k          2591   2656   1024   32    8 : tunables    0    0    0 : slabdata     83     83      0
kmalloc-512        13326  13696    512   32    4 : tunables    0    0    0 : slabdata    428    428      0
kmalloc-256        18044  18112    256   32    2 : tunables    0    0    0 : slabdata    566    566      0
kmalloc-192        20525  21000    192   21    1 : tunables    0    0    0 : slabdata   1000   1000      0
kmalloc-128         2240   2240    128   32    1 : tunables    0    0    0 : slabdata     70     70      0
kmalloc-96          4662   4662     96   42    1 : tunables    0    0    0 : slabdata    111    111      0
kmalloc-64         19925  20480     64   64    1 : tunables    0    0    0 : slabdata    320    320      0
kmalloc-32         57600  57600     32  128    1 : tunables    0    0    0 : slabdata    450    450      0
kmalloc-16         21248  21248     16  256    1 : tunables    0    0    0 : slabdata     83     83      0
kmalloc-8          13312  13312      8  512    1 : tunables    0    0    0 : slabdata     26     26      0
kmem_cache_node      576    576     64   64    1 : tunables    0    0    0 : slabdata      9      9      0
kmem_cache           384    384    256   32    2 : tunables    0    0    0 : slabdata     12     12      0
```

## The SLUB Allocator

We focus on the SLUB allocator only, as it is the default one. 

As we will see, in some sense, memory allocation in the kernel is simpler than in user mode, as the metadata (in general) is not stored along with the data itself (such as it happens in the userland heap) but in different structures. Below we will describe these objects.

Each CPU has an **active slab** of each given type. This means that, when allocating an object of a given type/size in a given CPU, space will be taken only from the active slab, until no more free space is left. When there is no more space, another slab will be marked as active for this CPU. Note that different processors have different active slabs.

*<u>Note</u>*: Previously, the `page` object had some fields to keep track of slab metadata (such as `void *freelist`, `short unsigned int inuse`, ...). These are referenced in some previou literature, but since v5.17-rc1 [they have been removed](https://github.com/torvalds/linux/commit/07f910f9b7295b6a28b337fedb56e612684c5659#diff-dc57f7b72015cf5f95444ec4f8a60f85d773f40b96ac59bf55b281cd63c06142) from this struct. This is metadata can be found in the `struct slab` (see below).

### `struct kmem_cache`

[This struct](https://elixir.bootlin.com/linux/v5.19.16/source/include/linux/slub_def.h#L90) is basically the abstraction of a slab cache as defined above. There is one instance of a `kmem_cache` for each object type that has a dedicated cache and for each size of the generic caches. It's used to manage all the slabs of the given object type/size.

Some important fields of this struct are the following:

- `const char *name`: The name of the slab cache (such as `kmalloc-32`, etc.)
- `unsigned int object_size`: The size of an object in the cache
- `unsigned int object_size`: The size of an object in the cache including metadata (if it has metadata)
- `struct kmem_cache_cpu __percpu *cpu_slab`: A pointer to a struct `kmem_cache_cpu` (see below). Actually, it's not a regular pointer but a `__per_cpu` pointer, which means that the address space is not the same (and some computation needs to be done to obtain the actual address of the `kmem_cache_cpu` object)
- `struct kmem_cache_node *node[MAX_NUMNODES]`: A pointer to an array of `kmem_cache_node` structs (see below)
- `unsigned int offset`: The offset within the objects where the pointer to the next free object is stored (when the object is in the freelist). See Freeing  Objects section below
- `unsigned long random`: Only present if `CONFIG_SLAB_FREELIST_HARDENED` is set, it is used to obfuscate the addresses of the pointers inside the freelist. See Freeing  Objects section below
- `unsigned int *random_seq`: Only present if `CONFIG_SLAB_FREELIST_RANDOM` is set. It's a random sequence so that the initial freelist (with all the objects of a slab, when a slab is created) is not sequential, but it has a random order. In other words, this means that objects won't be allocated sequentially at the beginning. See Freeing  Objects section below

*<u>Note</u>*: There is an exported variable named `slab_caches` which is a doubly-linked list with all the slab caches.

*<u>Note</u>*: There is also an exported variable named [`kmalloc_caches`](https://elixir.bootlin.com/linux/v5.19.16/source/include/linux/slab.h#L339) with all the kmalloc (generic) caches. It's a matrix where the first index indicates the [kmalloc cache type](https://elixir.bootlin.com/linux/v5.19.16/source/include/linux/slab.h#L320), which can be:

- `KMALLOC_NORMAL = 0`: corresponds to the caches named `kmalloc-8`, `kmalloc-16`, etc.
- `KMALLOC_CGROUP`: corresponds to the caches named `kmalloc-cg-8`, `kmalloc-cg-16`, etc.
- `KMALLOC_RECLAIM`: corresponds to the caches named `kmalloc-rcl-8`, `kmalloc-rcl-16`, etc.
- `KMALLOC_DMA`: corresponds to the caches named `dma-kmalloc-8`, `dma-kmalloc-16`, etc.

The second index indicates [the size of the objects](https://elixir.bootlin.com/linux/latest/source/mm/slab_common.c#L686) within this cache. Index `n` corresponds to `kmalloc-2^n`, which holds objects of the sizes from `2^(n-1)+1` to `2^n`.

### `struct kmem_cache_cpu`

[This struct](https://elixir.bootlin.com/linux/v5.19.16/source/include/linux/slub_def.h#L48) manages the active slab of the current CPU. Some important fields of this struct are:

- `void **freelist`: Pointer to the next available object in the slab. In other words, the next allocation in this slab will return the address pointed by this field
- `struct slab *slab`: Pointer to the active slab

### `struct kmem_cache_node`

[This struct](https://elixir.bootlin.com/linux/latest/source/mm/slab.h#L741) keeps track of partial slabs (i.e., slabs with free space) that are not active. Depending on the configuration, it also keeps track of full slabs. Some important fields of this struct are:

- `unsigned long nr_partial`: Number of partial slabs
- `struct list_head partial`: List of partial slabs
- `struct list_head full`: Only defined if `CONFIG_SLUB_DEBUG` is set, it's a list of full slabs.

### `struct slab`

[This struct](https://elixir.bootlin.com/linux/v5.19.16/source/mm/slab.h#L9) contains a slab's metadata, such as:

- `struct kmem_cache *slab_cache`: The slab cache (represented by a `kmem_cache`) this slab belongs to
- `void *freelist`: First free object in the slab

## Freeing Objects

When an object is freed with `kfree`, they are added to the slab freelist. The first object in the freelist is pointed by the `kmem_cache_cpu.freelist` field. The address of each subsequent object in the freelist is stored at offset `kmem_cache.offset` inside the previous free object, as can be seen tracking the calls from [`kfree`](https://elixir.bootlin.com/linux/v5.19.16/source/mm/slub.c#L4572) to [`set_freepointer`](https://elixir.bootlin.com/linux/v5.19.16/source/mm/slub.c#L381).

Some kernel options that can be enabled related to freelists are the following:

- **Hardened Freelist** (`CONFIG_SLAB_FREELIST_HARDENED`): When enabled, the pointers in the freelist [are obfuscated](https://elixir.bootlin.com/linux/v5.19.17/source/mm/slub.c#L327) XORing the original address (`ptr`) with a per-cache random number (`s->random`) and the address where the pointer is stored (`ptr_addr`):

  ```c
  /*
   * Returns freelist pointer (ptr). With hardening, this is obfuscated
   * with an XOR of the address where the pointer is held and a per-cache
   * random number.
   */
  static inline void *freelist_ptr(const struct kmem_cache *s, void *ptr,
  				 unsigned long ptr_addr)
  {
  #ifdef CONFIG_SLAB_FREELIST_HARDENED
  	/* ... */
  	return (void *)((unsigned long)ptr ^ s->random ^
  			swab((unsigned long)kasan_reset_tag((void *)ptr_addr)));
  #else
  	return ptr;
  #endif
  }
  ```

  If a vulnerability exists that could allow an attacker to overwrite these pointers, in order to achieve a successful attack they would need to know both `s->random` and `ptr_addr` (which could happen, but it's more difficult)

- **Freelist randomization** (`CONFIG_SLAB_FREELIST_RANDOMIZATION`): Introduced in v4.8, when this is enabled the initial order of the chunks in the freelist are no longer sequential. In other words, when the slab is created, all chunks are free and are thus part of the freelist. However, if this is enabled, allocations won't be in the same order that they are in the slab (first allocation will return the first chunk in the slab, and so on), but they will follow a random order. There is a per-cache random sequence (`s->random_seq`) that determines this order (defined [on cache initialization](https://elixir.bootlin.com/linux/v5.19.17/source/mm/slub.c#L1846)), and each slab in the cache [takes a random starting point within this sequence](https://elixir.bootlin.com/linux/v5.19.17/source/mm/slub.c#L2000) (so that allocations in different slabs won't start with the chunk at the same offset).

  This is was intended as a mitigation against **kernel heap overflows**, because before this if a vulnerable and a target objects were allocated one after the other, they were always adjacent, so it was easy to exploit it. However, this can be easily bypassed by allocating many target objects, then freeing one of them, and then allocating one vulnerable object. Indeed, the vulnerable object will be surrounded by target objects and then we can trigger the heap overflow and overwrite one of the target objects. There is [a great post](https://duasynt.com/blog/linux-kernel-heap-feng-shui-2022) by Michael S and Vitaly Nikolenko that explains this very well (section "FREELIST pointer randomisation")

## Inspecting kmalloc-caches with GDB

- Print the name of a given kmalloc-cache (e.g, kmalloc-32 corresponds to index 5 as 2^5 = 32):

  ```
  gdb> p kmalloc_caches[0][5]->name
  ```

- Print `struct kmem_cache_cpu` of the kmalloc-32 cache (management structure of the current CPU slab):

  ```
  set $current_cpu = $_thread - 1
  set $cpu_offset = __per_cpu_offset[$current_cpu]
  set $cpu_slab = (struct kmem_cache_cpu *)((void *)kmalloc_caches[0][5]->cpu_slab + $cpu_offset)
  p *$cpu_slab
  ```

## References

- [Linux Kernel Documentation on the Slab Allocator](https://www.kernel.org/doc/gorman/html/understand/understand011.html). While it is focused on the SLAB allocator (not SLUB), the definitions of the main concepts can be found here

- [The Slab Allocator in the Linux Kernel](https://hammertux.github.io/slab-allocator): Overview of the slab allocator, with details on SLAB, SLUB and SLOB. Also overview of the Buddy Allocator

- [The Slub Allocator](https://github.com/PaoloMonti42/salt/blob/master/docs/0x00_SLUB_refresher.md): Great summary of how the SLUB allocator works and its related structures

- [Linux kernel heap feng shui in 2022](https://duasynt.com/blog/linux-kernel-heap-feng-shui-2022): Great overview of some internals of the slab allocator and how they affect the exploitation, specifically cache aliasing,  the `SLAB_ACCOUNT` flag, hardened usercopy, changes around the `GFP_KERNEL_ACCOUNT` flag (which [kind of kill](https://twitter.com/bienpnn/status/1516314186938470402) the `msg_msg` spray technique in newer kernel versions), and freelist pointer randomisation

- [Looking at kmalloc() and the SLUB Memory Allocator](https://ruffell.nz/programming/writeups/2019/02/15/looking-at-kmalloc-and-the-slub-memory-allocator.html)

- [The Linux kernel memory allocators from an exploitation perspective](https://argp.github.io/2012/01/03/linux-kernel-heap-exploitation/)

  

