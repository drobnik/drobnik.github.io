---
layout: post
title:  "Why the world needs memblock simulator"
date:   2022-01-28 09:02:01 +0100
categories: post
tags: linux memblock outreachy
image: assets/img/socialpreview.jpg
---

After a very hectic time full of kernel patches and moving boxes
<span id="back-1">[<a href="#foot-1">1</a>]</span>, I sent
[the first version of memblock simulator to linux-mm](https://lore.kernel.org/linux-mm/cover.1643206612.git.karolinadrobnik@gmail.com/).
Now, it’s a good time to explain what is this all about and why such thing is needed in the first place.

# What?

We could say that “simulator” is a fancy word to describe a test suite that uses the actual memblock code.
This program runs in user space (i.e., outside the kernel), which causes a problem in itself.
Memblock uses a bunch of kernel definitions, which are unavailable here.
Compilation results in >100 errors and many more warnings. Still, we have to create an
illusion that all the structures and functions are present. This means one
thing – work out all dependencies, stub required definitions and make the compiler happy. And that’s what I did.

After getting the memblock running, I was able to move on to the test cases. If you take a
look at the patches, you can see it’s a series of unit tests exercising different memblock functions.
Define a region (or more), try to add/reserve/remove/free it and check if different memblock data
structures get updated to expected values. It’s quite simple. At least for now.

# Why?

[Like I mentioned before](https://insecuremode.com/post/2021/12/14/getting-to-know-memblock.html),
memblock is quite a strange beast. It performs memory management before the actual memory allocators
are initialized, which is very early in the booting process. This makes testing and debugging
it difficult. There were a couple of regressions that happened
in the past <span id="back-2">[<a href="#foot-2">2</a>]</span><span id="back-3">[<a href="#foot-3">3</a>]</span>,
and maybe they could be avoided if there was an automated way of testing memblock-related changes.
For now, my project makes sure that the basic memblock API behaves as expected.

# The future

The next thing I plan to work on is adding test coverage for `memblock_alloc_*` and
`memblock_phys_alloc_*` functions. They are responsible for finding a suitable memory region that can
be used for allocation. Testing these will need some prep work, because we  wish to work on real,
valid memory ranges. Why is that, you may ask.

In its basic form, memblock can store 128 entries of available and reserved memory regions.
There are a couple of cases when this is not enough and resizing either of the arrays is required.
We don’t need to look far to find an example - on x86, UEFI can return a memory map that has more
regions than what memblock can support<span id="back-4">[<a href="#foot-4">4</a>]</span>.
So, what would happen if we were to test this use case as it is now? We could do something like this:

- Allow array resizing (call `memblock_allow_resize()`)
- Register some memory as available (call `memblock_add(...)`)
- Try to add/reserve `INIT_MEMBLOCK_REGIONS` +  1.

The last region addition [would trigger the array resize](https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L643).
This function, `memblock_double_array`, looks for a free spot based on what was added to `memblock.memory`
and `memblock.reserved`. Now, the question is – what memory block did we register as available?
What is the base address? `0x0`? `0xaabbcc`? Either way, we can be certain it’s not a valid address
for this program. Even if `memblock_double_array` finds space for the resized array within this range,
it segfaults on `memcpy` called [here](https://elixir.bootlin.com/linux/latest/source/mm/memblock.c#L464).

So, for now, the solution is to use valid memory ranges in `memblock_add()` returned by `malloc`.
It’s to be seen if this method will work for testing the allocation functions. If it does, you’ll
see an appropriate patchset in a month or so.


# The far future

In the big picture, the memblock simulator should be able to not only test its features one by one,
but use them all together. For example, we could pass (or generate) a physical memory
layout to the simulator<span id="back-5">[<a href="#foot-5">5</a>]</span>, perform usual memblock tasks,
and simulate releasing the memory to the buddy page allocator. Such a test would check if the final
memory map was correctly initialized.

Still, it’ll take a lot of time to get there. Unfortunately, the 5 weeks I have won’t be enough to implement all of this.

<hr/>

<ul class="footnotes">
<li>
    <div id="foot-1">
    [1] – I know, organizing a move in the middle of the internship is an <i>interesting</i> idea…
    <a href="#back-1">⤴</a>
    </div>
</li>

<li>
    <div id="foot-2">
    [2] – <a href="https://lore.kernel.org/lkml/20210726192311.uffqnanxw3ac5wwi@ivybridge/">
    Regression bisected to fa3354e4ea39 (mm: free_area_init: use maximal zone PFNs rather than zone sizes), linux-mm lore thread
    </a>
    <a href="#back-2">⤴</a>
    </div>
</li>

<li>
    <div id="foot-3">
    [3] – <a href="https://bugzilla.kernel.org/show_bug.cgi?id=213073">
    Bug 213073 - kernel panic early in boot on xeon x5690 dual core
    </a>
    <a href="#back-3">⤴</a>
    </div>
</li>

<li>
    <div id="foot-4">
    [4] – <a href="https://elixir.bootlin.com/linux/latest/source/arch/x86/kernel/e820.c#L1309">
    e820__memblock_setup
    </a> implementation
    <a href="#back-4">⤴</a>
    </div>
</li>

<li>
    <div id="foot-5">
    [5] – Which is a challenge in itself…
    <a href="#back-5">⤴</a>
    </div>
</li>
</ul>
