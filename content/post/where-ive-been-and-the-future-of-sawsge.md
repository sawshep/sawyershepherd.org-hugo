---
title: "Where I've Been and the Future of Sawsge"
date: 2023-01-22T18:56:22-05:00
---

In case you didn't notice, my website has been offline for a
good while---since Christmas 2022, I believe. Additionally,
it now looks like this...

![New front page](/images/2023-01-22-front-page.jpg)

While it used to look like this...

![Old front page](/images/2022-05-27-front-page.png)

I've migrated from building my site with my own custom
static site generator
[Sawsge](https://github.com/sawshep/sawsge) and hosting it
on my own server to building it with
[Hugo](https://gohugo.io/) and hosting it on Github pages.
I'm surprised I've stooped so low as to put my website on
someone else's servers (Microsoft's nonetheless!) and
further contributed to the centralization of the internet,
but I have.

See, I came home from visiting my family after Christmas to
a flooded house. Despite leaving the heat on and the taps
dripping, my pipes froze and burst, causing my ceiling to
cave in. Many of my possessions were damaged, including my
3D printer. Luckily, my instruments are mostly okay---My
cajon is caked in drywall dust, my acoustic guitar smells
bad, and the bridge of my electric guitar is all rusty, but
everything is still playable.

I've been living in temporary housing for a while, so my
home server is MIA. I'll most likely rebuild it on a NUC
with NixOS I was able to recover, so it might be months
before it goes back up on my own hardware.

Don't take this transfer as my abandonment of Sawsge,
either. Hugo is okay, and it makes many hard things easy,
but there are just as many things that should be easy but
are still very hard. In fact, these flaws with Hugo are the
exact reason I built Sawsge; I hoped to build an SSG with
the good features Hugo has while adding my own (admittedly
opinionated) improvements. It's the classic circle of FOSS
software, isn't it? :grin:

My experience with Hugo has also made me step back and
evaluate the whole architecture of Sawsge. I'd like to port
something like the Go templates Hugo uses to Sawsge, but I'd
have to make my own DSL in Ruby. Unfortunately, that isn't
the easiest to do in Ruby, but it is very easy to do in
Lisp! That's right, I've been thinking of porting Sawsge to
Lisp. I may do it under a different name and leave Sawsge as
the archived legacy Ruby version to avoid making too many
breaking changes to any projects still using it.

Anyways, I'm going to try to post more often. Most of my
posts thus far have been documenting my technical projects,
but I think I'll branch out into my other hobbies
(specifically music) and perhaps write some opinion pieces.
I've got a post about AI art from my perspective as an
artist and a techie in the works.
