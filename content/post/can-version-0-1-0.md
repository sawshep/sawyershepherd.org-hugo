---
title: "Can Version 0.1.0"
date: 2022-05-01T00:00:00-05:00
---

I have released my program Can on RubyGems and the Arch User
Repository.

## Introduction

I have a feeling this post is going to get hairy with a
program name like *Can*!

Can is an idea for a program I've had for a very long time.
As the FreeDesktop.org [trash
specification](https://specifications.freedesktop.org/trash-spec/trashspec-latest.html)
says, "An ability to recover accidentally deleted files has
become the de facto standard for today's desktop user
experience." But what about a user's *command-line*
experience? I'm sure we've all accidentally annihilated a
file or two or three or four with rm. I know I certainly
have, and after irreversibly deleting an entire unpublished
project of mine, I set out to make a tool that would bring
the de facto experience of the desktop to the command line
to prevent this from ever happening again.

## Concept

Put simply, Can is nondestructive rm. Instead of unlinking
files, Can moves them into a hidden trash directory (by
default `~/.local/share/Trash/files`) and creates a metadata
file for later recovery.

Option parity with rm is a primary goal for the program.
Unlike rm, however, Can is not intended for use in scripts.
At the moment, it has a few blocking confirmation prompts to
prevent data deletion--option parity only provides a
smoother user experience.

While it is not intended for use in scripts, having similar
options to rm allows for a smoother user experience.

## Development

My initial idea was to program Can in C. In theory, it's a
very simple program--just move a file to a trash directory
and create a metadata file. In practice, the intricacies
that come with making a tool with a user-friendly CLI made
using C a headache.

I also experimented with the idea of writing Can in Bash by
stringing together existing shell commands, but there was
too much required functionality to implement easily. I would
have to fill in some big cracks with my own scripting, and I
hate writing in Bash. Due to my frustration, I shelved the
program for many months.

During a school break, inspiration to finish the program
struck. This time, I settled on Ruby because of my previous
frustrations with slow development speed. With Ruby's
expressiveness, I quickly implemented basic features,
achieved partial option parity with rm, and set up
continuous delivery to
[RubyGems](https://rubygems.org/gems/can_cli) and the
[AUR](https://aur.archlinux.org/packages/can) (tutorial on
how to do Ruby CD coming soon!).

## Usage

Here are a few examples of how to use Can:

Action | Command
---|---
Move files to the trashcan | `can foo.txt bar.txt`
Move files and directories to the trashcan | `can -r foo.txt bar.d`
List files in the trashcan | `can -l`
List files in the trashcan that match a regex pattern | `can -l '^foo'`
View `.trashinfo` metadata of select files in the trashcan | `can -n foo.txt bar.d`
Untrash files | `can -u foo.txt bar.d`
Empty select files from the trashcan | `can -e foo.txt bar.d`
Empty everything in the trashcan | `can -e`

Can has many safety nets built in to prevent data deletion.
Trashing files of the same name has been safely handled to
prevent overwriting data. Additionally, Can will not untrash
a file to a path that already exists. And just like rm, Can
has an `-i` option to wait for user confirmation before
trashing a file.

To see the full usage options, run `can --help`.

## Faults and Future Plans

### Performance

A program like Can that is most likely to be run often on
small groups of files benefits most from startup time
optimization. Choosing Ruby may have been a good idea to
boot programmer productivity, but it's awful for startup
times.

Frankly, I'm not if there's any simple solutions to slow
startup times. The nuclear option is to port the entirety of
the program to [Crystal](https://crystal-lang.org/). To put
it simply, Crystal seeks to be compiled Ruby. My description
does not do this very neat language justice, so please see
the homepage for yourself!

### Test-Driven Development

Can runs against no tests. I know, I know, it's a terrible
developer practice. I've been working to find a testing
method I like. So far, the best option seems to be running
[Minitest](https://rubygems.org/gems/minitest/versions/5.15.0)
tests with a [Rake](https://github.com/ruby/rake) task on
build.
