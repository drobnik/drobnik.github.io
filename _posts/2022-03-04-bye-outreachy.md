---
layout: post
title:  "atexit(bye_outreachy);"
date:   2022-03-04 16:30:04 +0100
categories: post
tags: personal outreachy memblock
image: assets/img/socialpreview.jpg
---

Today is the last day of my Outreachy internship. So, it’s a perfect time to take a moment and reflect on how this project went and what was it like for me to develop it.

To start with some facts, I submitted ~~25~~ 26 memblock simulator patches ([1](https://lore.kernel.org/linux-mm/cover.1643796665.git.karolinadrobnik@gmail.com/), [2](https://lore.kernel.org/linux-mm/cover.1646055639.git.karolinadrobnik@gmail.com/),
[3](https://lore.kernel.org/linux-mm/20220304125249.253578-1-karolinadrobnik@gmail.com/)), totalling 3951 additions (+) and 105 deletions (-). The number of lines is this high mostly because of the ASCII diagrams in test descriptions, like this one
[here](https://lore.kernel.org/linux-mm/1c0ba11b8da5dc8f71ad45175c536fa4be720984.1646055639.git.karolinadrobnik@gmail.com/#iZ31tools:testing:memblock:tests:alloc_nid_api.c). If everything goes well, my work will be merged into v5.18 (which is mid-May, if I'm not mistaken).

Just like every other project, memblock simulator took some turns, and there were bumps in the road. Thankfully, I was able to get memblock running in the user space, and add some tests on the top of it. I had a lot of milestones in my project timeline, but life went in the way, as it usually does. Personal stuff took away five working days from my internship. Also, I forgot about one important thing when putting my plan together. Memblock actually supports _two_ memory allocation directions, not just one! This meant that I had to implement x1.5 more test cases than I initially anticipated. In addition to this, I decided to reduce code duplication and refactor some checks, so they could be used in both directions. It was an extra week of work alone. But hey, I did my best to stay on track and squeeze in as many tests as possible, and that’s what counts!

Coming back to getting the memblock running, I’m quite happy with how fast I was able to stub the definitions. The plan was to have a working project skeleton by 23rd December. On that exact day, I sent the first version of memblock simulator to my mentor. Surprising, isn’t it? Still, while I was right when it came to estimating the time needed to implement something, I underestimated the effort required to make my patch set upstream-ready. I don’t know why, but I kept forgetting about how much time was needed to address reviewers’ comments, write a cover letter and respond to emails. I hope I’ve learned my lesson by now.

There’s one big thing I hoped to get done. Yes, I’m talking about the memblock allocation blog post. I decided to skip a couple of Outreachy blog posts and write a big, technical one. Unfortunately, researching the implementation and writing my thought down ate up all the time I put aside for writing. Priorities of this project were clear, so I had to come back to implementing tests. I hoped that I’ll find an hour or two to work on it ad-hoc. Unfortunately, it wasn’t until the final week that I had time to get back to it. The good thing is that even trying to write this post helped me a lot when implementing `memblock_alloc()` tests. I knew its insides, had some notes written down, so writing the checks was a pretty smooth process.

On a very personal note, I wanted to say that it was quite a special internship for me. Before the contribution period began, the confidence about my skills and capabilities was very low<span id="back-1">[<a href="#foot-1">1</a>]</span>, and I was worried about my future as a programmer. Fast-forward to today, I feel better about my code, `git send-email` is not that anxiety inducing, and I’m knowledge-hungry again. A great improvement, I must say! Also, I met a bunch of smart, lovely people, learned a ton of things and had plenty of interesting conversations. For this opportunity, I’m forever grateful.

My Linux kernel adventure doesn’t end here. Even better, I’ll be able to continue working on it full-time. It’s not in memory management per se, but I might touch some DRM GEM things…

The memblock simulator project isn’t going away, either. It’ll be back for
[the upcoming Outreachy round](https://www.outreachy.org/apply/project-selection/#linux-kernel), how exciting! The next person will have a chance to improve and extend the project I started. If you’re one of the applicants, and you’d love to dip your toes into memblock stuff, just ping me. I’ll be happy to help.


<hr/>

<ul class="footnotes">
<li>
    <div id="foot-1">
    [1] – It wasn’t even an impostor syndrome, I was the impostor!
    </div>
</li>
