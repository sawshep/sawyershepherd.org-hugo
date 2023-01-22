---
title: "Fall 2021 KSU HS CTF Writeup"
date: 2021-11-13T00:00:00-05:00
---

[I placed
third](https://github.com/AndyGreenPhD/HS_CTF/tree/main/fall_2021)
at the end of the challenge, winning myself $50. This is how
I solved the hardest and least-solved questions of each
category.

## Introduction

I've been trying to work on my resume recently, and what
better way to do it than with a fun CTF organized by a local
college? The competition ran from 10 A.M. to 3:30 P.M. on
Sunday, November 13th. There were 33 applicants (including
myself), 26 of which competed.

## Log Analysis

> What is the total size of the objects returned to client
> IP address 66.249.73.135? (in bytes)

The provided file was a large Nginx log with many
different clients requesting many different resources,
whether they existed or not.

I ran `grep "^66.249.73.135 LogFile.txt" > requests.txt` to
get a file of requests only by the target IP.

Each entry in an Nginx log follows this format:[^format]

```
ip - user [time] "request" status bytes "referer" "user_agent"
```

I can use regex capture groups to get the number of bytes
transferred, turn it into an `int`, and add it to a total.

I wrote a Perl script to do just that:

```perl
# Open the list of requests made
open my $lines, "requests.txt" or die "Can't open file";

my $total = 0;

# For each line in the file,
while ( my $line = <$lines> ) {
    # if the line matches this regex, capture the match in
    # parentheses...
    if ($line =~ /66.249.73.135 - - .* "GET .*" [0-9]* ([0-9]*) ".*/s) {
        # ...and add it to the total.
        $total += int($1)
    }
}
# Print the total
say $total
```

## Network Traffic Analysis

> What is the flag?
> The flag is in the format `KSU-***-*****`

The attached file was a `.pcapng` file.

I uploaded the file to [A-Packets](https://apackets.com/),
an online pcap analyzer. In the 'Images' section, I found an
image with the flag written on it.

![An image of the host of Ancient Aliens. The caption says,
"The flag is `KSU-2021-BEBE`"](/images/network-traffic-analysis.jpg)

## Cryptography

> Given the string below, see if you can find the flag using
> the attached file:
> ÖÊËÕËÕÏÛÈÃØÈÑÐÖ

The attached file was a font. I pasted the given string into
LibreOffice and set it to the attached font, which revealed
the flag: `THISISMYFAVFONT`

## Password Cracking

> [WoWProgress](https://www.wowprogress.com/) is a site
> which tracks the rankings of the top guilds in the world
> for the game World of Warcraft. The following is an md5
> hash of the name of one of the top 100 guilds in the world
> for the "Sanctum of Domination" tier.  Crack the hash and
> submit the name of the guild as your answer! 
>
> `39d8b865c6ba1c6cfd354a2a1763c1ff`

I simply copied the top 100 guilds by column into a wordlist
and ran it in `john` against the hash to get the flag:
`Wizards and Monkeys`.

## Python Programming

Unfortunately, I ran out of time on the hardest question of
this section.

## Forensics

All the questions in this category were not written by the
organizers, but instead provided to them by CTFd. In my
opinion, all the forensics challenges were undervalued at
only 100 points each.

### A Friend

I was the only participant to solve this question!

> My friend has been acting a little weird lately. He sent
> me a weird message. Can you figure out what's up?

Attached is a zip file named `a_friend.zip`.

First, I attempted to unzip the file, which returned an
error. By running `binwalk` on the file, I saw that, while
indeed there was a zip file in it, there were also other
embedded files.

```
DECIMAL  HEXADECIMAL  DESCRIPTION
0        0x0          Zip archive data, at least v2.0 to extract, compressed size: 12250, uncompressed size: 24626, name: a friend.docx
12404    0x3074       End of Zip archive, footer length: 22
12426    0x308A       PNG image, 5312 x 2988, 8-bit/color RGB, non-interlaced
15889    0x3E11       Zlib compressed data, default compression
```

I extracted these all at once. Inside, there was...

* `0.zip`: A copy of the original zip file. Strange, I
  know---in fact, I think this is what caused `binwalk` to
  behave irregularly.
* `3E11`: An empty file.
* `3E11.zlib`: Zlib compressed data.
* `a friend.docx`: A Word document.

Notice how there is no PNG image present as `binwalk`
previously suggested there would be. Unfortunately, I did
not notice this until later.

The Word document had white text that blended into the
background. It described that somewhere in the archive the
friend had hidden his/her location.
    
I unzipped the `.docx` file---they're really just zip
archives---and perused around for any sort of location to no
avail. I decided to take a step back; I used `strings
a_friend.zip | less` on the whole archive to check for
anything I might have missed. In fact, I did miss
something---EXIF data for the supposed PNG embedded in the
file:

```
exif:GPSAltitude="220/1"
exif:GPSLatitude="41,32.888585N"
exif:GPSLongitude="90,29.983900W"
exif:GPSTimeStamp="2016-02-06T15:08:22Z"
```

As it turns out, this was a red herring. Different map
applications read these coordinates differently, and no
coordinates or addresses were accepted as the flag.

I inspected the output of `binwalk` again and finally
noticed the missing image supposedly in the file. I
extracted the image individually from the file as opposed
extracting all the other files at once---it worked, and the
flag was written across the image.

![An image of 2 cute rats in a cage at a pet store. The
caption says,
"`flag{help_im_stuck_at_the_pet_store}`"](/images/a-friend.png "A
Friend Flag")

[^format]: https://rtcamp.com/tutorials/nginx/log-parsing/
