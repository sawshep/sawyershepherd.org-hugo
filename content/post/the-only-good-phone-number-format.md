---
title: "The Only Good Phone Number Format"
date: 2025-02-15T21:03:41-05:00
---

## TL;DR

Use +1 (555) 123-4567 ext. 890. It's immediately
identifiable as a phone number and encodes semantic
information in its format, including the area code, exchange
code, and line number.

## Introduction

Good UX should minimize cognitive load on the user. It seems
phone number formats missed that memo--I read and type a lot
of phone numbers for my job, and I can say that nearly all
formats suck.

There are two main problems with formats:
1. Not immediately, unambiguously recognizable as a phone number
2. Easy to lose place in the number while reading--little to
   no visual blocking

So, a good format is both immediately, unambiguously
recognizable as a phone number, and easy to read. It seems
simple, but let's take a look at a number of formats I've
seen in the wild.

## Standards

### The Rogue Dot Dot Dot

1.555.123.4567

This one isn't really a standard format, but I do see it
used often.

This one looks too much like an IP address. You can tell it
isn't with a quick glance, but you shouldn't *need* a quick
glace! 

### E.164

+15551234567

This standard specifies that the country code, area code,
and subscriber number should be lumped together. It's not
readable, and contains about as much semantic meaning as the
name of the standard, and it's not immediately obvious it's
a phone number. Let's take a look at another number:

+1447851987654

How many digits is that? If you can't tell just by looking
at it, it is not a good format. It's 13, by the way.

### E.123

+1 555 123 4567

This one is better. There is finally some visual separation
between each section with the spaces. It's a little clearer
that it's a phone number. However, because only spaces
separate the sections of the number, it's easy to lose your
place when you are looking back and forth between it and a
phone.

### RFC 3966

tel:+1-555-123-4567

Well, at least this one tells you it's a telephone number,
and you can click it if you have an application that
supports opening numbers! All the hyphens are ugly, though.
It's useful to link to, but there should be some text in
place of the number, the same way you might write
[sawyershepherd.org](https://sawyershepherd.org) but
actually link to [https://www.sawyershepherd.org](https://www.sawyershepherd.org).

## The Champion

+1 (555) 123-4567

Wow, look at that. Nicely sectioned out and labelled for
maximum semantic meaning, including the country code, area
code, exchange code, and subscriber line. A variety of
separators--a plus sign, parentheses, spaces, and dashes--so
you don't lose your place.

Allegedly, this format is actually documented by in E.164
and the North American Number Plan (NANP) as a national
format, but I was unable to find a first-party source to
support this.

## Non-U.S. Numbers

The Champion is definitely optimized for NANP numbers. Sure,
the meaning contained within the format may not be the exact
same across different countries, but that is no excuse to
use a terrible format! The hyphen between the exchange code
and the subscriber line can be adapted to separate long
strings of digits for better readability.

Let's take the French number +33 1 23 45 67 89. It's not
immediately recognizable, and while the pairs of numbers are
nice for readability, there are too many spaces and it's
easy to lose your place when reading.

In The Champion format, it would be +33 (1) 2345-6789. I
happen to think that's much more readable. Now, the French
people probably disagree with me, but you know what?

![A man standing in front of a crowd. He is saying "Yes, you are all wrong"](/images/you-are-all-wrong.jpg "Yes, you are all wrong")
