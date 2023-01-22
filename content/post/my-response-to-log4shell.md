---
title: "My Response to Log4shell"
date: 2022-02-28T00:00:00-05:00
---

I have remodeled my entire server stack to better defend
against critical vulnerabilities.

I know this blogpost is long overdue, with the waves
Log4Shell sent through the security world having mostly
passed, but I promise my action was timely!

## Background

Previously, my server ran Debian 11 (Bullseye) with a custom
sandboxing SystemD configuration for every service I ran (read
more about my old setup [here](/post/2021/11/06)). This
approach offered performance benefits at the extreme cost of
my time---it takes ages to work out all the kinks in a
custom configuration.

Unfortunately, when Log4Shell was publicized, the
only service I had left to sandbox was the only service I
ran to be affected by the vulnerability.

I heard of the vulnerability a few hours after the news
broke. While my logs held no indication of malicious actors,
I knew that one can never be too careful if an attacker
potentially has full RCE capabilities, and promptly wiped my
server.

## Rebuilding

Left with a blank slate, I decided that I needed to take a
smarter approach to my server infrastructure. After the
scare of Log4Shell, security, reproducibility, and
ease-of-setup took precedence over minute performance
optimizations. I took to putting each of my services in
Docker containers, which I managed with Ansible on top of
Fedora Server as my distribution of choice.

In front of all the containers sits a reverse proxy to
forward requests to the appropriate container. This also
further fortifies my network in combination with UFW---no
connections can get in or out of the containers without
going through the reverse proxy.

Overall, I am satisfied with this new software stack. It has
proven to be magnitudes more dependable and secure than my
previous infrastructure. 

## Future Plans

If Fedora ever becomes an inconvenience to me, I will switch
to NixOS. I have [a friend in the Nix
cult](https://wizardwatch.net) who has given it great praise
for its reproducibility and declarative package management.
NixOS in combination with Nix Flakes may even allow me to
completely remove Ansible from the stack.

Another problem that may come to annoy me in the future is
that my server has no webhook capabilities. As of now, if I
want to make a post to my blog, I have to:

1. Write the post
2. Commit and push the post
3. Make a new build with
[Sawsge](https://github.com/sawshep/sawsge)
4. Run the Ansible playbook to update the site on my server

I'd prefer to just push changes to the repository,
triggering the Git server to send a webhook to my server,
which would generate a new Sawsge build right on the server.
It'll be a pain to set up, but will save me labor in the
long run.
