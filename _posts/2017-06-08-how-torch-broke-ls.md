---
layout: post
title: How Torch broke ls and made me vulnerable (or the hidden dangers of LD_LIBRARY_PATH)
category: general
description: If used incorrectly, LD_LIBRARY_PATH can cause a plethora of issues
---

So, today I decided to check out the firmware of a wireless router using the amazing utility
known as [`binwalk`](https://github.com/devttys0/binwalk). Nothing too unusual and everything seemed to work quite smoothly:

![Imgur](http://i.imgur.com/gyjiEwD.png)

Then I went to `cd` into the newly-extracted firmware directory, and that's where it all went wrong...

<!--description-->

![Imgur](http://i.imgur.com/1ryNc4J.png)

Immediately, I could tell something was up. `Powerline` was no longer working and I couldn't
see anything resembling a bash prompt. I sheepishly typed in `ls` and saw _this_:

![Imgur](http://i.imgur.com/FGsT0Em.png)

How was this possible? My system was working just fine a second ago! I quickly
attempted to run `strace` to see if I could find out what was going on and hoped it'd help
figure out what was happening here...

![Imgur](http://i.imgur.com/UbaNd53.png)

Sure enough, for some odd reason I was trying to load `libpthread` from the **current directory**
rather than from `/usr/lib64` where it normally resides on my machine. A quick check from the
file command and Caja validated that the issue was that the firmware I was extracting just
_happened_ to have a rogue `libpthread.so.0` sitting in the top directory:

![Imgur](http://i.imgur.com/TvVXpAZ.png)

Moving back up a directory, of course, fixed the problem. Now the question that was on my
mind was _"WHY ON EARTH IS MY WORKSTATION TRYING TO LOAD LIBRARIES FROM $PWD?"_

A quick inspection of my LD_LIBRARY_PATH gave me:
```
/home/joshua/torch/install/lib:/home/joshua/torch/install/lib:/home/joshua/torch/install/lib:
```
Which _seemed_ okay. [Torch](http://torch.ch/) created that when it wrote to my .bashrc and nothing seemed
particularly out of the ordinary. But nonetheless, unexporting it "miraculously" fixed
my problem and I was able to browse the directory of the router's extracted firmware.

After a bit of research and playing around, I came across a startling fact:
On many machines, Glibc's `ld.so` will add the **current working directory** to the library search
path on any LD_LIBRARY_PATH with a trailing colon. That means that one little `:` at the end
was causing every executable that requires any shared library to start its search in the current
directory that I started it in. Every git repository that I downloaded and ran a trusted command in,
every directory I extracted from a tar file, every disk image that I mounted had the ability to
inject a malicious shared library that `ld.so` would have happily loaded for the application that
required it. And because I installed Torch for machine learning work over a year ago, I had been
completely vulnerable to this type of attack for a long time! I went to the Torch codebase to confirm
that it was really the culprit, and sure enough:

![Imgur](http://i.imgur.com/igVJqDk.png)

Because my LD_LIBRARY_PATH was empty before I installed Torch, the export in `torch-activate`
results in a rogue colon being placed at the end of it, resulting in potential issues such as this.

Any users of the Torch machine learning library should make sure their LD_LIBRARY_PATH is secure...
