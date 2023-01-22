---
title: "How I Started My Server"
date: 2021-11-06T00:00:00-05:00
---

My server used to be an old gaming desktop, now running
Debian 11. I publish my blog with a custom static site
generator written in Ruby.

## Hardware

This website is running off an old midrange gaming computer,
from circa 12 years ago. I received it for free from a
friend, who himself also got it for free from an elderly
neighbor.

The specs are as follows:

* MSI K9N6PGM2-V2 Motherboard
* AMD Athalon II X2 250
* NVIDIA GeForce 8600 GT
* 2x2GB 400MHz DDR2 Memory

The computer had no drive, so I bought a 256GB SATA SSD and
slapped it in. The case doesn't have mounts for 2.5" drives,
so the SSD just kinda... floats around. It's not much of a
problem though, since the server stays on a shelf all the
time.

It runs great, except for one unusual issue: I can't get it
to restart automatically after a power outage. Even after
enabling "Restart After AC Power Loss" in the BIOS, it
doesn't work! I'll have to check for a newer BIOS version to
flash.

## Software

### Operating System

After a great deal of consideration, I chose to install
Debian 11 (Bullseye) on the server. I usually opt for
SystemD-less distros, but I'll concede that the init system
makes it a breeze to create and sandbox services. Debian is
a staple in the server world too---it's fast, simple, and
~~outdated~~ stable. And with my experience using Debian for
desktop and Armbian for server, there was no better choice.

### Web Server

The server is running Nginx with a custom sandboxing
configuration. Here's the service override file, in case
anyone is interested.

Note: You must change the PID file in
`/etc/nginx/nginx.config` to `/run/nginx/nginx.pid` for this
to work.

```systemd
[Service]
User=www-data
Group=www-data
RuntimeDirectory=nginx
CapabilityBoundingSet=CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_BIND_SERVICE
PIDFile=/run/nginx/nginx.pid
NoNewPrivileges=true
LockPersonality=true
SystemCallFilter=@system-service
SystemCallFilter=~@privileged @resources
PrivateTmp=true
ProtectSystem=full
ProtectProc=invisible
ProtectHome=true
ProtectHostname=true
ProtectClock=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectKernelLogs=true
ProtectControlGroups=true
MemoryDenyWriteExecute=true
SystemCallArchitectures=native
ProcSubset=pid
PrivateDevices=true
RestrictSUIDSGID=true
RestrictRealtime=true
RestrictNamespaces=true
RemoveIPC=true
RestrictAddressFamilies=AF_INET AF_INET6
UMask=0077
```

It makes the master process not root and prevents Nginx from
doing things that it doesn't need to do.

I know that's a bit of a cop-out of an explanation, sorry.
If you want to learn more about SystemD sandboxing, the
[manpage for
`systed.exec`](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)
is very good. You can also use `systemd-analyze security` to
see a qualitative score of all your services, and
`systemd-analyze security [service]` to see how to harden a
specific service. The configuration above gives a exposure
score of 1.5---pretty good for a internet-facing service.

### Static Site Generator

I build my website with a [static site generator I wrote in
Ruby](https://github.com/sawshep/sawsge). I could go on
forever about why I love Ruby, but that's a blogpost for
another day. The rundown is that it's a beautifully
ergonomic, flexible language that allows for insane
productivity.

Wait, that sounds a lot like Python too, right? Well, that's
another future blogpost. Python had a lot of weird little
gremlins that like to surface in the middle of a project. My
initial attempt at a static site generator was in Python---I
only got about 10 lines in when I realized that Python can't
handle paths with dots in the middle, e.g.
`src/./post/2021/11/4/index.html` isn't an openable file.
You have to jump through hoops to turn it into an expanded
path.

That's how Python usually is for me---jumping through hoops
to do something that should be simple. There's a lot of that
in Python, and that's just not a developer experience I want
from a supposedly ergonomic scripting language.

\</rant>

The basic goal of my webserver is to reduce boilerplate.
Each source post is a Pandoc Markdown fragment.

Because Markdown is a superset of HTML, I can write a source
post entirely in HTML if I really need fine-grain control,
and it'll still be valid Markdown.

This is how the source directory of the site usually looks
like. The generator is hyper-specific to this directory
hierarchy, but it could easily be modified to fit another.

```
src/
├── footer.html
├── header.html
├── index.md
├── post
│   └── 2021
│       └── 11
│           └── 6
│               └── index.md
└── style.css
```

For every Markdown file in the `post/` dir, the generator
determines the date from the path, converts the fragment to
HTML with Pandoc, adds the header and footer, then
automatically fills out tags like the title or date. The
final generated files are compliant with XHTML 1.0 Strict.

The homepage generation follows a different approach to that
of the posts. The title and a collapsible summary of each
post is appended to the homepage in order of date, with the
summary of the latest post expanded.

This is what the output directory would look like if I ran
the above source directory through the generator:

```
out/
├── index.html
├── post
│   └── 2021
│       └── 11
│           └── 6
│               └── index.html
└── style.css
```
