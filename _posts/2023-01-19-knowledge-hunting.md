---
layout: post
title:  "Hunting for knowledge in Linux kernel"
date:   2023-01-19 17:37:14 +0100
categories: post
tags: linux practices
image: assets/img/socialpreview.jpg
---

Linux documentation is patchy sometimes. Reading it can be a frustrating experience, especially when you're just starting out.
But fear not, there is plenty of information available! It's just about knowing where to start digging for it.

During the last year, I found a couple of things that made learning about Linux kernel features (and its dark
corners) easier. Some of them weren't obvious, so I thought I could share them here.

# Cover letters

When a developer wishes to contribute their changes to the kernel, they have to send them as a series of patches to the mailing list. Most of the time, the patches are accompanied by a cover letter – an introductory email that explains the reason for
the changes. It's a great starting point for learning about a specific feature or design change, as it usually provides
some background information and, if you're lucky, links to the previous work in the area.

So, how do we find cover letters? Given that you know the name of a data structure/function that was introduced (or changed) in
the patchset<span id="back-1">[<a href="#foot-1">1</a>]</span>, and have the source code checked out locally:

1. Locate the file with the definition
2. Run `git blame` to find the related commit (it might not be the latest one!)
3. Keep going back in time with blame until you find a commit that could be part of the bigger series
4. Take the first line of the commit (so-called **commit subject**) and search for an email that introduced it. If you're not subscribed to a specific mailing list, search mailing list archives, for example `lore.kernel.org` or
   `patchwork.freedesktop.org`<span id="back-2">[<a href="#foot-2">2</a>]</span>
5. Go up in the thread to find the first email, and voilà, you found the cover letter!

The cover letters usually target general audience (i.e. technical, but not working on the feature), so given you
have basic understanding of the code area, you should be able to understand them. If not, then they may provide some pointers on
what you should look up first. Because of this, I highly recommend checking out the cover letters first, even before reading commit messages.

One important thing to mention here. Before you start reading the cover letter you found, make sure you're looking at
**the latest** version of the series to have an accurate picture of the changes you're reviewing. Oftentimes the authors
have to update their patches given the review comments, so there will be more than one version of the series sent to the
mailing list. On the first search you might find an older version which wasn't merged to the kernel, so keep searching
until you find the accepted one.

# Commit messages

Linux kernel folks pay attention to the commit messages<span id="back-3">[<a href="#foot-3">3</a>]</span>, and they are still
an essential source of knowledge in the kernel. After you've familarised yourself with the cover letter, you should
review the commit messages of patches in the series. They may only describe "before" and "after" of the patch (especially if it's
a bugfix), but you can also find some links or, even [snippets of dmesg logs](https://lore.kernel.org/all/20220922195127.2607496-1-nathan@kernel.org/) associated with it.

When reading the messages, you may notice a convention of using **tags**. Usually they are used to indicate who worked on the
patch<span id="back-4">[<a href="#foot-4">4</a>]</span> (`Signed-off-by`) or have accepted/reviewed it
(`Acked-by`/`Reviewed-by`), but there are tags that provide more context, like links to the related mailing list discussions,
bug reports or even articles. The [Linux kernel documentation](https://docs.kernel.org/process/submitting-patches.html#describe-your-changes) mentions some of them, but there are a couple more I've seen in the wild:

- `Link` - a URL link to related discussion (usually a thread in the mailing list archive), background information, bug report
- `Fixes` - points to the commit that introduced a bug
- `Closes` - link to the issue the commit fixes. What tracker is linked here differs between subsystems, it could be Bugzilla or Gitlab issues tracker ([example](https://gitlab.freedesktop.org/drm/intel/-/issues))
- `References` - saw it used to mention a different commit in the tree, but I'm not quite sure if it's still in use. In the past it seemed to be used the same way as `Link`

# Presentations and blogs

Due to the hectic nature of the kernel development, much of the information about the changes is first shown at the conferences.
Unfortunately, this doesn't always translate to the formal documentation. The good news is that many of the talks are recorded
and you can watch them on Youtube. If you haven't stumbled upon them while looking for the
cover letter, it's worth to do a dedicated search for them. Some of the conferences that you should look up are [Linux Plumbers](https://www.youtube.com/@LinuxPlumbersConference), Open Source Summit and Embedded Linux Conference on [The Linux Foundation channel](https://www.youtube.com/@LinuxfoundationOrg) and [FOSDEM](https://www.youtube.com/@fosdemtalks) (with this one being
less focused on the Linux kernel itself).

Not a fan of video-learning? Many presenters share their slides, so you can try looking for these. Some of them are
pretty good on their own, but from my experience, they usually lack the context of the live presentation.

Speaking of written materials – blogs posts are also a great source of knowledge. Less formal than documentation, but might
provide more background than the cover letters. Authors might write about their experiences on personal as well as their
company's blogs.

You can also find articles written by kernel developers on [LWN.net](https://lwn.net/). If you haven't heard about
this page, think of it as a "news" site that reports on what's happening in the
Linux kernel<span id="back-5">[<a href="#foot-5">5</a>]</span>. Some of the newest articles are subscribers-only, but they become
available to the public after two weeks or so. But, if you're really passionate about the project, consider buying subscription
to support them.

# Books

This might seem like an odd suggestion, but I think that even outdated books can help in learning about the kernel. There's one
caveat – look up the recent source code and compare it with what is presented in the book. Sometimes the differences are small
enough that you can fill the gaps on your own. Other times, you might have to dig deeper. But even then, the book gives you some
keywords that you can use while looking up commits or cover letters.

As for reading the code, you can do it locally or use [Elixir code browser](https://elixir.bootlin.com/linux/latest/source).
Elixir is an online version of the Linux source with all definitions, usages and references. I'm in love with this tool – it's
very convenient, and allows switching between kernel versions. Nothing stops you from comparing the [latest changes in the file](https://elixir.bootlin.com/linux/latest/source/drivers/gpu/drm/i915/gem/i915_gem_context.c)
with [something from the past](https://elixir.bootlin.com/linux/v5.13/source/drivers/gpu/drm/i915/gem/i915_gem_context.c).

As for the books, I don't have any insightful recommendations, but I can share what I know or read:

- [Linux Device Drivers](https://lwn.net/Kernel/LDD3/) is the "classic" I'm working through right know. While reading it, I try to keep an eye on recent code examples.
- During my Outreachy internship, I read a substantial portion of [Linux Kernel Development](https://www.goodreads.com/en/book/show/8474434-linux-kernel-development). It's a great introduction, gives enough detail to understand what's going on without overwhelming the reader. Like I said, the code is outdated, but I read the source code alongside the book, and I was able to get a lot from studying it. I skipped the chapters describing how to compile the kernel or send a patch to the mailing list, so I can't say how helpful they are.
- A recommendation from my Outreachy mentor was [Understanding The Linux Virtual Memory Manager](https://www.kernel.org/doc/gorman/). I read some of it and it looked pretty good.

# IRC?

There is one place that I think I should mention when talking about knowledge hunting – IRC. Once you read countless articles,
presentations, commit messages or whatever is out there, you may want to talk to an actual human being. Although it's possible to
ask for help or clarification on the mailing list, it's more common to see folks coming to IRC to chat with devs. I'm just a
lurker, so I can't talk from my experience, but I saw people getting answers to their questions as well as with their patches.
It's not a 24/7 helpline (at times, quite contrary), but with a small dose of patience, you can get the help you need.

I heard that nowadays people hang out on #linux-rt channel for general discussions. The active channels I'm aware of are #linux-mm (Memory Management subsystem) and #dri-devel (Direct Rendering Manager subsystem), all of that hosted on
[oftc.net](https://oftc.net/) network.

If you're not familiar with IRC or you're just terrified of it<span id="back-6">[<a href="#foot-6">6</a>]</span>, you can check
out [this guide](https://libera.chat/guides/) before heading straight to the chat. It describes the basics of IRC, how to connect
and use the network.

# Conclusion

As you can see, we're not in the dark as it appeared at first glance. Thanks to the mailing list archives, we can trace the
development history of different kernel features or bug fixes. With the help of presentations, books and the source code itself,
we can begin to understand what is going on in the Linux kernel. With enough time and patience, you may be able to contribute and
fix something. Yes, I agree, it is still hard to piece all the information together, but... that's part of the fun! Give
yourself time and keep going deep into the rabbit holes. Explore obscure details. Be a Linux detective, and thoroughly enjoy it!

<hr/>

<ul class="footnotes">
<li>
    <div id="foot-1">
    [1] - I know, that's usually the trickiest part. Here, I'd either use google and read documentation (sic!) to find some keywords I can search with
    <a href="#back-1">⤴</a>
    </div>

    <div id="foot-2">
    [2] - Patchwork is not just a mailing list mirror per se, <a href="https://github.com/getpatchwork/patchwork#patchwork">more info</a>
    <a href="#back-2">⤴</a>
    </div>

    <div id="foot-3">
    [3] - ...most of the time...
    <a href="#back-3">⤴</a>
    </div>

    <div id="foot-4">
    [4] - this tag doesn't just indicate a patch author. It can be also used for someone who helped with getting this patch delivered to the kernel, or altered it in a significant way before sending it to the mailing list
    <a href="#back-4">⤴</a>
    </div>

    <div id="foot-5">
    [5] - posting about other open source projects from time to time
    <a href="#back-5">⤴</a>
    </div>

    <div id="foot-6">
    [6] - I was!
    <a href="#back-6">⤴</a>
    </div>
</li>
