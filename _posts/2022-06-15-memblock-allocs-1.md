---
layout: post
title:  "Unraveling memblock allocs, Part 1"
date:   2022-06-15 09:31:04 +0100
categories: post
tags: linux memblock
image: assets/img/memblock_glitch.jpg
---

When you take a look at the memblock allocation API for the first time, you might feel a bit overwhelmed. You see a bunch of functions
with "memblock_alloc" or "memblock_phys_alloc" prefixes, coupled together with all possible combinations of start/end addresses
and NUMA node requirements that you can think of. But this is not a surprising view, given the fact that memblock has to
accommodate the needs of ~20 different architectures, from Motorola 68000 through PowerPC to x86 and ARM.

It’s possible to look into the abyss of allocs and understand what are they about. As we will see later on, both groups of
allocation functions, `memblock_alloc*` and `memblock_phys_alloc*`<span id="back-1">[<a href="#foot-1">1</a>]</span> use the same logic to find free memory block. Although this
logic is a bit complicated, it makes sense and can be broken down into smaller, easy to digest, parts. After learning the way how
it works, we can apply this knowledge to _all_ functions. We would just need to remember what memory constraints we can ask for
and where.

There’s a lot to take in, so we need to approach this topic with caution. Descending deep down into macros like [`__for_each_mem_range_rev`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/memblock.h#L203) straight away won’t teach us much about the flow of the allocation process. We have to be able to zoom in and out
 when needed. Because of this, I decided to break this analysis down into two parts. In the first one, I’d like to
 show you how `memblock_alloc*` and `memblock_phys_alloc*` functions converge and discuss the general flow of this
 common part. Once we understand how it works, we can move on to the second part. There, we’ll look at the implementation
 of the core allocation function and see what it does to grant memory at (almost) all costs. We’ll dissect its internal
 parts, delve into details, and get to see quite a lot of helper functions.

Let’s start our analysis with taking a look at two functions -
[`memblock_alloc`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/memblock.h#L424) and [`memblock_phys_alloc_range`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1423). As you may
recall from the introductory post on memblock<span id="back-2">[<a href="#foot-2">2</a>]</span>, the first one represents the
group of functions that return virtual addresses to the allocated memory regions, whereas the second one operates on physical
addresses.

# memblock_alloc

One of the most widely used boot time memory allocation functions.
`memblock_alloc` is used across different architectures to
allocate memory for:
- OS data structures like [MMU context managers](https://elixir.bootlin.com/linux/v5.18.3/source/arch/ia64/mm/tlb.c#L62), [PCI controllers](https://elixir.bootlin.com/linux/v5.18.3/source/arch/alpha/kernel/pci.c#L395) and [resources](https://elixir.bootlin.com/linux/v5.18.3/source/arch/x86/kernel/e820.c#L1160)
- String literals, like [resource names](https://elixir.bootlin.com/linux/v5.18.3/source/arch/alpha/kernel/core_marvel.c#L84)
- [Error info logs](https://elixir.bootlin.com/linux/v5.18.3/source/arch/ia64/kernel/mca.c#L380)
- and more

A quick glance at its implementation reveals a quite short function – it's a wrap for a call
to [`memblock_alloc_try_nid`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1586):

{% highlight c %}
static __always_inline void *memblock_alloc(phys_addr_t size, phys_addr_t align)
{
    return memblock_alloc_try_nid(size, align,
        MEMBLOCK_LOW_LIMIT, MEMBLOCK_ALLOC_ACCESSIBLE, NUMA_NO_NODE);
}
{% endhighlight %}

The third and fourth parameter instruct memblock to use the whole address range covered by the entries of the
[`memblock.memory`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L112)
array. `memblock_alloc`
doesn’t care about which NUMA node will be used here. To signal this, [`NUMA_NO_NODE`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/numa.h#L14) value is passed in.

Going one step deeper doesn’t reveal much of `memblock_alloc`
inner workings. `memblock_alloc_try_nid` is yet another wrapper
that calls [`memblock_alloc_internal`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1474) and clears the memory pointed by `ptr` if one was allocated:

{% highlight c %}
void * __init memblock_alloc_try_nid(
            phys_addr_t size, phys_addr_t align,
            phys_addr_t min_addr, phys_addr_t max_addr,
            int nid)
{
    void *ptr;
(...)
    ptr = memblock_alloc_internal(size, align,
                       min_addr, max_addr, nid, false);
    if (ptr)
        memset(ptr, 0, size);

    return ptr;
}
{% endhighlight %}

Maybe third time's a charm? Well, `memblock_alloc_internal` turns out to be yet another wrapper. But it’s a more elaborated one:

{% highlight c %}
static void * __init memblock_alloc_internal(
                phys_addr_t size, phys_addr_t align,
                phys_addr_t min_addr, phys_addr_t max_addr,
                int nid, bool exact_nid)
{
     phys_addr_t alloc;

    ① if (WARN_ON_ONCE(slab_is_available()))
         return kzalloc_node(size, GFP_NOWAIT, nid);

    ② if (max_addr > memblock.current_limit)
         max_addr = memblock.current_limit;

    alloc = memblock_alloc_range_nid(size, align, min_addr, max_addr, nid, exact_nid);

    ③ if (!alloc && min_addr)
    ④    alloc = memblock_alloc_range_nid(size, align, 0, max_addr, nid, exact_nid);

    ⑤ if (!alloc)
         return NULL;

     return phys_to_virt(alloc);
}
{% endhighlight %}

In ①, we check if `memblock_alloc` and similar wrappers were called when the slab allocator was available<span id="back-3">[<a href="#foot-3">3</a>]</span>. If this is the case, the function uses slab allocation instead of the memblock one and returns.
Moving on, ② checks if the requested maximum address is above the memblock’s memory limit, and caps it if needed. With all of
this from the way,
[`memblock_alloc_range_nid`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1354),
 the core allocation function, is called. This function returns the base _physical_
address of a memory block on success. In the next step ③, `memblock_alloc_internal`
checks if the allocation was successful. If this wasn’t the case, and we had a minimal address specified
(e.g. was passed via [`memblock_alloc_from`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/memblock.h#L438) helper), it calls
 `memblock_alloc_range_nid` without the lower address requirement in ④. The check in ⑤ makes sure that we got a valid memory
 location before trying to map it to a virtual address (i.e. a void pointer) in `phys_to_virt`. Given the `memblock_alloc_range_nid`
 call succeeded, `memblock_alloc_internal` returns a virtual memory address to an allocated memory chunk, returned by
 `memblock_alloc` at the end of the call chain.

# memblock_phys_alloc_range

You’ve might have expected to see [`memblock_phys_alloc`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/memblock.h#L407)
discussed here. Its name contrasts well with `memblock_alloc` name, the
function itself has a simple signature, and it looks convenient to use. It turns out that it’s _the least popular_ function in
the phys_alloc group. It's not that interesting anyway, it’s just a wrapper for
[`memblock_phys_alloc_range`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1423):

{% highlight c %}
static __always_inline phys_addr_t memblock_phys_alloc(phys_addr_t size,
                               phys_addr_t align)
{
    return memblock_phys_alloc_range(size, align, 0,
                     MEMBLOCK_ALLOC_ACCESSIBLE);
}
{% endhighlight %}

Actually, this one is the most widely used when working with physical addresses. `memblock_phys_alloc_range`
function is used to allocate memory for:
- Crash kernel logs ([1](https://elixir.bootlin.com/linux/v5.17.1/source/arch/mips/kernel/setup.c#L455),
 [2](https://elixir.bootlin.com/linux/v5.17.1/source/arch/riscv/mm/init.c#L1006), and more)
- [NUMA distance table](https://elixir.bootlin.com/linux/v5.17.1/source/arch/x86/mm/numa.c#L379)
- [reallocated initrd](https://elixir.bootlin.com/linux/v5.17.1/source/arch/x86/kernel/setup.c#L267)

As you can see, x86 makes an extensive use of `memblock_phys_alloc*` functions. Interestingly enough, the word has it that
this family of functions is getting deprecated ([source](https://www.youtube.com/watch?v=NP7wU7A218k&t=1006s)).

Coming back to our function, this is how `memblock_phys_alloc_range` looks like:

{% highlight c %}
phys_addr_t __init memblock_phys_alloc_range(phys_addr_t size,
                         phys_addr_t align,
                         phys_addr_t start,
                         phys_addr_t end)
{
    memblock_dbg(...);
    return memblock_alloc_range_nid(size, align, start, end, NUMA_NO_NODE,
                    false);
}
{% endhighlight %}

That's correct, we've discovered yet another wrapper. But, wait a second. We saw `memblock_alloc_range_nid` function before,
didn’t we? Indeed, we saw it just a minute ago in `memblock_alloc_internal`. This is the common part I’ve mentioned in the
beginning of this post. In fact, the allocation functions that return physical addresses just wrap a call to `memblock_alloc_range_nid` with specific parameters. Here, we have no preference for NUMA node, and pass the same value as we did
in `memblock_alloc`.

To sum this up in a visual way, the call chains we’ve followed here can be seen this way:

<img src="{{site.url}}/media/img/memblock/common_part.png" alt="a diagram showing how two alloc functions indirectly call memblock_alloc_range_nid">

The same pattern emerges for other allocation functions like [`memblock_phys_alloc_try_nid`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L1448)
or [`memblock_alloc_low`](https://elixir.bootlin.com/linux/v5.18.3/source/include/linux/memblock.h#L446). It turns out that
`memblock_alloc_range_nid` is called, both directly and indirectly, by a group of functions, where some of them do additional
checks or clean-ups. You don’t need to take my word for it – feel free to take a look at the memblock source code to convince
yourself that this is the case.

# The big picture

Now, we can agree that all roads lead to `memblock_alloc_range_nid`. One more thing you need to know about it is that this
function is very determined to grant memory. It is ready to drop memory requirements passed by the programmer and retry multiple
times to allocate memory before giving up.

So, what does the general flow of this function look like? In essence, `memblock_alloc_range_nid` does two things – it first
looks for an available memory region , and then tries to reserve it. Searching for a free memory block is broken down into a
multiple helpers and macros, each of them implemented for both memory allocation directions – top-down and bottom-up. The former
starts allocating from high addresses (that is, from the end of the memory), whereas the latter looks at the low addresses first.
Although bottom up feels more intuitive to think about, most architecture use the top-down direction.

Finding an available memory spot is aided by a low-level helper, which returns the base address of the region on success. If
everything goes well, `memblock_alloc_range_nid` reserves a region and a new entry in
 [`memblock.reserved`](https://elixir.bootlin.com/linux/v5.18.3/source/mm/memblock.c#L117) is created. If not, the
function checks if it can loosen up the memory requirements, like a preference for a specific NUMA node, and tries to allocate memory
again. This approach works in most cases, so a free region is found and reserved on the next attempt. If we’re out of luck, no
memory gets allocated, and the function returns `0` address to indicate the failure. This process can be pictured as the
following:

<img src="{{site.url}}/media/img/memblock/nid_fun_flow.png" alt="a diagram showing how memblock_alloc_range_nid checks for the memory allocation direction to find a proper set of helpers and how it retries with less constraints on failure">

# What's next

That’s it for the high-level tour of memblock allocs. Now, you can see how all the functions we thought were doing different
things are actually pretty close to one another. We also discussed the flow of the core allocation function,
`memblock_alloc_range_nid`,  but many questions remain unanswered. What exact memory constraints can be scrapped when memblock
retries allocation? How are memory regions iterated in the top-down direction? Or do we care about memory mirroring at all? To
answer these, we have to take a close look at the implementation of `memblock_alloc_range_nid` and analyse what bits it uses down
the way. As it’s a lot of work, let’s save it for the part 2 of our investigation.

# References

1. Memblock implementation – [source](https://elixir.bootlin.com/linux/v5.17.1/source/mm/memblock.c), [header](https://elixir.bootlin.com/linux/v5.17.1/source/include/linux/memblock.h)
2. 🎥 [Boot Time Memory Management, OSS EU 2020](https://www.youtube.com/watch?v=NP7wU7A218k)

<hr/>

<ul class="footnotes">
<li>
    <div id="foot-1">
    [1] - it's a catch-all names for functions in one group, read it as a regular expression
    <a href="#back-1">⤴</a>
    </div>

    <div id="foot-2">
    [2] - for a refresher, go <a href="https://insecuremode.com/post/2021/12/14/getting-to-know-memblock.html"> here
    </a>
    <a href="#back-2">⤴</a>
    </div>

    <div id="foot-3">
    [3] - many architectures free the memblock data structure in <code>mm_init</code>, which is also where
    the slab allocator gets initialized. This check is to protect a programmer from using a
    non-existing, at this point, memblock allocator
    <a href="#back-3">⤴</a>
    </div>
</li>
