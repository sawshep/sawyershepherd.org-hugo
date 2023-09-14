---
title: "UNG Cyberhawks \"Memory\" Writeup: Linux Memory Forensics with Volatility 2"
date: 2023-09-13T01:00:47-04:00
draft: false
---

This post serves as a guide to installing and using
Volatility 2 on Linux memory dumps, as well as a walkthrough
for the UNG CyberHawks' "Memory" challenge. If you're
looking for a Volatility 3 tutorial, I'd suggest following
[this
writeup](https://www.pkusinski.com/sekai-ctf-2022-writeup-symbolicneeds/)
by Pawel Kusinski. My writeup will not cover the creation of
Linux OS profiles---for this, you can follow [the official
documentation](https://github.com/volatilityfoundation/volatility/wiki/Linux#creating-a-new-profile).

If you'd like to follow along with this guide, you can
download [the memory dump
here](https://1drv.ms/u/s!AsnToTHSN8cqmcxPx_fZlmT-h6gLRA)
and [the OS profile
here](https://1drv.ms/u/s!AsnToTHSN8cqmcxN25B1a7GbzLjJxw).
Our goal is to find the following:
1. The filename of the flag file
2. The full path of the flag file
3. The flag itself

## Setting Up the Environment

I used a Kali virtual machine for this challenge because I
didn't want to sully my machine with Python 2! We must first
update our repositories before installing Python 2, Pip 2,
and the dependencies for Volatility:

```bash
$ sudo apt update
$ sudo apt-get install python2
$ curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py
$ sudo python2 get-pip.py
$ sudo apt-get install python2-dev
$ python2 -m pip install setuptools
$ python2 -m pip install pycryptodome
$ python2 -m pip install distorm3
```

Now we can clone the repository and run Volatility:

```bash
$ git clone https://github.com/volatilityfoundation/volatility.git
$ cd volatility
$ python2 vol.py --help
```

Let's set up our profile. We are provided with a pre-made
profile, which we need to pass to Volatility in a certain
way. Create a directory for the profile and move the profile
there:

```bash
$ mkdir -p plugins/overlays/linux
$ mv Ubuntu_Profile.zip plugins/overlays/linux
```

Now if we pass the plugins directory to Volatility, our
profile will show at the top of the list:

```bash
$ python2 vol.py --plugins=plugins/ --info
Profiles
--------
LinuxUbuntu_Profilex64 - A Profile for Linux Ubuntu_Profile x64
VistaSP0x64            - A Profile for Windows Vista SP0 x64
VistaSP0x86            - A Profile for Windows Vista SP0 x86
...
```

We will use this profile for all Volatility commands:
```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem [plugin]
```

## Basic Forensics

We shouldn't get ahead of ourselves by jumping straight in
to using Volatility---performing basic forensics with
programs like Strings and Binwalk can help understand the
problem on a deeper level. Unlike Volatility, both programs
are included in Kali by default.

### Strings

By searching for "flag" through the output of `strings
dump.mem`, we can find multiple references to `flag.png`.
Additionally, we encounter this:

```cpp
file.read(buffer.data(), BUFFER_SIZE); #include <vector>
#include<unistd.h> using std::cerr; using std::cout; using
std::endl; using std::ifstream; using std::ios; using
std::streamsize; using std::vector; const int BUFFER_SIZE =
1000; ifstream file("flag.png", ios::binary); int main() {
vector<char> buffer(BUFFER_SIZE); vector<char> file_content;
while (file) { "tmp/prog.cpp" 44L, 816C written 18,0-1 8%
file.read(buffer.data(), BUFFER_SIZE); {ent.size() << end
```

I've split this across multiple lines to make it a little
easier to read. It looks like a `vi` buffer of a C++ program
intended to load the flag image file into memory. Note that
the vector initial buffer size is only 1,000 bytes, much
smaller than any image would be. In C++, vectors are
dynamically allocated, meaning that loading the image into
the vector will not cause an overflow, but rather reallocate
and copy the buffer in whatever way is implemented by the
standard library. Knowing this will help recover the image
later. We would have never found this important detail had
we not performed basic forensics first!

### Binwalk

Let's see if Binwalk can find any PNG's from the dump:

```bash
$ binwalk dump.mem | grep -i png
58663184 0x37F2110 PNG image, 37 x 47, 8-bit/color RGBA, non-interlaced
58665056 0x37F2860 PNG image, 20 x 25, 8-bit/color RGBA, non-interlaced
58665872 0x37F2B90 PNG image, 20 x 24, 8-bit/color RGBA, non-interlaced
58666424 0x37F2DB8 PNG image, 38 x 45, 8-bit/color RGBA, non-interlaced
58667992 0x37F33D8 PNG image, 38 x 47, 8-bit/color RGBA, non-interlaced
58669880 0x37F3B38 PNG image, 24 x 21, 8-bit/color RGBA, non-interlaced
...
```

Binwalk returns that it found 84 PNGs. I attempted to
extract them, but I quickly ran out of storage space on my
VM as Binwalk's extractions are often a significant portion
of the binary size. This is because the program usually
extracts files from the header to the end of the binary,
leaving on a lot of junk data. For this challenge, manually
combing through all the files Binwalk extracted would have
taken too much time---it can be solved in a much more
intelligent and efficient way.

## Finding the Filename

With basic forensics out of the way, we can get on to using
Volatility. Let's take the most straightforward approach to
find the filename. We will use the `linux_lsof` Volatility
plugin (you can find a full list of plugins with the
`--info` flag) to list all open files:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_lsof
...
0xffff9d3e002d4a40 readFile 2812 0 /dev/pts/1
0xffff9d3e002d4a40 readFile 2812 1 /dev/pts/1
0xffff9d3e002d4a40 readFile 2812 2 /dev/pts/1
0xffff9d3e002d4a40 readFile 2812 3 /home/admin/tmp/flag.png
...
```

Towards the end, we have the answer to part two of the
challenge, the full path of the flag:
`/home/admin/tmp/flag.png`.

## Attempting to Read the Flag via the Inode

Now that we know the path of the flag, we can attempt to
read it from memory. We will use the `linux_find_file`
plugin in two parts to do so. First, we need to find the
inode of the file:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_find_file --find="/home/admin/tmp/flag.png"
Inode Number Inode              File Path
------------ ------------------ ---------
     2148886 0xffff9d3e002c53f0 /home/admin/tmp/flag.png
```

Now, with the inode `0xffff9d3e002c53f0`, we can attempt to
extract the file from memory:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_find_file --inode=0xffff9d3e002c53f0 --outfile=flag.png
```

But there's an issue! If we `xxd` the file, we'll find that
it's all zeroes. This tripped me up; for a while I assumed
that the flag had not actually been zeroed, but instead that
I was using the command incorrectly. So, why is the flag
file zeroed?

### Reading Command Line History

Reading the command line history could give us more clues as
to why the flag file has been zeroed. We can use the
`linux_bash` plugin to do so:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_bash
Pid  Name Command Time                 Command
---- ---- ---------------------------- -------
2161 bash 2023-04-09 20:18:25 UTC+0000 vim /home/admin/tmp/prog.cpp 
2742 bash 2023-04-09 20:19:01 UTC+0000 g++ /home/admin/tmp/prog.cpp -o /home/admin/tmp/readFile
2742 bash 2023-04-09 20:19:03 UTC+0000 cd tmp/
2742 bash 2023-04-09 20:19:04 UTC+0000 ls
2742 bash 2023-04-09 20:19:09 UTC+0000 ./readFile 
2815 bash 2023-04-09 20:19:12 UTC+0000 rm flag.png 
2815 bash 2023-04-09 20:19:16 UTC+0000 cd ..
2815 bash 2023-04-09 20:19:19 UTC+0000 sudo ./avml/target/x86_64-unknown-linux-musl/release/avml ./dump.mem
2815 bash 2023-04-09 20:19:19 UTC+0000 er
```

The Bash history reveals the answer: the flag was removed
after it was loaded into memory, which means we can't
retrieve it the usual way.

## Finding Process Memory Mappings

So---because we were unable to recover the flag from the
inode, our last remaining avenue is to extract the PNG file
directly from a heap dump of `readFile`. In order to dump
the heap of `readFile`, we must identify the process ID and
the memory address of the heap. There are a number of
Volatility plugins that can retrieve process information; we
will use `linux_pslist` and search for "readFile":

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_pslist | grep readFile
Offset             Name     Pid  PPid Uid  Gid  DTB                Start Time
------------------ -------- ---- --------- ---- ------------------ ----------------------------
0xffff9d3e002d4a40 readFile 2812 2742 1000 1000 0x0000000007010000 2023-04-09 20:19:25 UTC+0000
```

With `readFile`'s PID of 2812, we can now find its memory
mappings with the `linux_proc_maps` plugin:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_proc_maps --pid 2812
Offset             Pid  Name     Start              End                Flags Pgoff   Major Minor Inode   File Path
------------------ ---- -------- ------------------ ------------------ ----- ------- ----- ----- ------- ---------
0xffff9d3e002d4a40 2812 readFile 0x000055dcf6b71000 0x000055dcf6b72000 r--       0x0   252     5 2114342 /home/admin/tmp/readFile
0xffff9d3e002d4a40 2812 readFile 0x000055dcf6b72000 0x000055dcf6b74000 r-x    0x1000   252     5 2114342 /home/admin/tmp/readFile
0xffff9d3e002d4a40 2812 readFile 0x000055dcf6b74000 0x000055dcf6b75000 r--    0x3000   252     5 2114342 /home/admin/tmp/readFile
0xffff9d3e002d4a40 2812 readFile 0x000055dcf6b76000 0x000055dcf6b77000 r--    0x4000   252     5 2114342 /home/admin/tmp/readFile
0xffff9d3e002d4a40 2812 readFile 0x000055dcf6b77000 0x000055dcf6b78000 rw-    0x5000   252     5 2114342 /home/admin/tmp/readFile
0xffff9d3e002d4a40 2812 readFile 0x000055dcf8978000 0x000055dcf89bd000 rw-       0x0     0     0       0 [heap]
0xffff9d3e002d4a40 2812 readFile 0x00007ff5a313e000 0x00007ff5a3142000 rw-       0x0     0     0       0 
0xffff9d3e002d4a40 2812 readFile 0x00007ff5a3142000 0x00007ff5a314f000 r--       0x0   252     5  818190 /usr/lib/x86_64-linux-gnu/libm-2.31.so
...
0xffff9d3e002d4a40 2812 readFile 0x00007ff5a36c1000 0x00007ff5a36c2000 rw-   0x2d000   252     5  818184 /usr/lib/x86_64-linux-gnu/ld-2.31.so
0xffff9d3e002d4a40 2812 readFile 0x00007ff5a36c2000 0x00007ff5a36c3000 rw-       0x0     0     0       0 
0xffff9d3e002d4a40 2812 readFile 0x00007ffcba570000 0x00007ffcba591000 rw-       0x0     0     0       0 [stack]
0xffff9d3e002d4a40 2812 readFile 0x00007ffcba5cf000 0x00007ffcba5d3000 r--       0x0     0     0       0 
0xffff9d3e002d4a40 2812 readFile 0x00007ffcba5d3000 0x00007ffcba5d5000 r-x       0x0     0     0       0 [vdso]
```

This command gives us the memory mappings of the main
process and its dependencies. We'll take down the address of 
`0x000055dcf8978000` for the next step.we can dump the heap with the

## Dumping the Heap

With the heap starting address in hand
(`0x000055dcf8978000`), we can use the `linux_dump_map`
plugin to dump the heap for closer inspection:

```bash
$ python2 vol.py --plugins=plugins/ --profile=LinuxUbuntu_Profilex64 -f dump.mem linux_dump_map --vma=0x000055dcf8978000 --dump-dir=.

```

We now have the heap of `readFile` dumped into the file
`./task.2812.0x55dcf8978000.vma`. If we examine hexdump of
the file using `xxd`, we can find four idential PNG headers.
Here's one:

```
$ xxd task.2812.0x55dcf8978000.vma
...
00015520: 8950 4e47 0d0a 1a0a 0000 000d 4948 4452  .PNG........IHDR
00015530: 0000 029c 0000 00ef 0806 0000 005f 6204  ............._b.
00015540: e300 0000 0662 4b47 4400 ff00 0000 0033  .....bKGD......3
00015550: 277c f300 0000 0970 4859 7300 002e 2300  '|.....pHYs...#.
00015560: 002e 2301 78a5 3f76 0000 0007 7449 4d45  ..#.x.?v....tIME
00015570: 07e7 0409 1135 1773 1be6 e600 0000 1974  .....5.s.......t
00015580: 4558 7443 6f6d 6d65 6e74 0043 7265 6174  EXtComment.Creat
00015590: 6564 2077 6974 6820 4749 4d50 5781 0e17  ed with GIMPW...
000155a0: 0000 2000 4944 4154 78da eddd 797c 4d77  .. .IDATx...y|Mw
000155b0: fef8 f177 ec41 548c c6d2 0ca2 4585 b10e  ...w.AT.....E...
000155c0: 4d27 a34d b5a9 56a5 640c 632c a5d4 3a89  M'.M..V.d.c,..:.
000155d0: 2199 9020 4aad 4150 7b8b d2d2 d652 3aa8  !.. J.AP{....R:.
000155e0: a6a8 8e69 a9b4 9950 6a19 c9d8 634b 4aa2  ...i...Pj...cKJ.
000155f0: 892c b29c ef1f 7ef1 1392 7b3f e7de 736f  .,....~...{?..so
00015600: 6e72 5fcf c723 8f07 c9e7 2cf7 7cce b9e7  nr_..#....,.|...
...
```

Let's extract each of these to their own files so we can
analyze each independently. We can use `dd` to copy a byte
slice from the heap dump, changing the `skip` and `count`
options relative to each header. Note that `xxd` uses
hexadecimal, while `dd` uses decimal. This example copies
from the first PNG header to just before the second PNG
header:

```bash
$ dd if=task.2812.0x55dcf8978000.vma of=png1.png bs=1 skip=87328 count=4112
```

Then, we repeat this for the other three PNGs. Finding and
converting the header offsets within the hexdump is left as
an exercise to the reader!

## Analyzing the PNGs

After carving four almost-PNGs from the heap, we can begin
to analyze them. The first step is to simply look at one.
Opening one of the files in an image viewer shows that they
look like this:

![Half of the flag image. Only the top few pixels of text
are visible.](/images/ung-cyberhawks-memory-writeup-linux-memory-forenics-with-volatility-2/flag-half.png)

Getting closer! Only the top few pixels of the flag are
visible, but it's good progress. Note that all the PNGs look
the same, despite each being a different size. Let's examine
the hexdump. The output is too long to paste here, but it
shows that each PNG is identical to the others up to a
point, and that the next PNG contains the previous. In other
words, the fourth and final PNG is the most complete of the
lot.

The second noticeable trait is there are 1,000 byte blocks
of zeroes between every 1,000 bytes of data. Remember that
the vector buffer size is 1,000 bytes. In C++, vectors are
dynamically expanded to fit. The exact methods depend on the
compiler and libraries used. The `readFile` program
specifically uses GCC and GNU's libstdc++. In this
implementation of vectors, if more data is stored into the
vector than is allocated, the vector will allocate a new
buffer double the size of the previous, copy the data, and
free the previous buffer. This is why much of the PNG data
is repeated throughout the buffer. I'm not exactly sure why
the 1,000 byte gaps of zeroes appear throughout the data,
though I believe it's an effect of memory management
optimization implemented for vectors.

## Rebuilding the PNG

After analyzing how data was stored in the vector on the
heap, we can reconstruct the PNG in a hex editor--I used
GHex. We'll edit the fourth PNG as it's the most complete.
By removing the 1,000 byte blocks of zeroes, we can slowly
reveal more of the flag. Here's the flag after removing one
block:

![Half of the flag image. The tops of a few letters are
discernible.](/images/ung-cyberhawks-memory-writeup-linux-memory-forenics-with-volatility-2/flag-more.png)

Now, we'll continue removing blocks of zeroes until the flag
is readable in its entirety.

![The fully readable flag image. It says
"SKY-EWVN-7427".](/images/ung-cyberhawks-memory-writeup-linux-memory-forenics-with-volatility-2/flag.png)
