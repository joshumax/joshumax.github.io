---
layout: post
title: Running DOOM on a Hacked CaptionCall Landline
category: general
description: An adventure on embedded device modification
---

![The CaptionCall](https://i.imgur.com/DsAbWMu.jpg)

Recently, I came across a very interesting telephone service called [CaptionCall](https://captioncall.com/). For those 
of you who don't know about this service, it's essentially a government-funded telephone captioning service that allows
deaf or hard-of-hearing individuals to see a textual description of audio calls. But, more interesting than the service
itself is the device they provide to their customers. It's a fancy, Linux-based touchscreen landline running a proprietary
UI atop a buildroot userspace. It's also quite capable, sporting an ARMv7 i.MX6 Quad SoC, 4GB of NAND, and a whopping 1GB!
of DDR3. Naturally when finding this out, I just had to get DOOM running on it.
Unfortunately for us, however, CaptionCall has no developer documentation for their device nor a way to access
a shell on the machine by default. Additionally, they provide no source code to their modified kernel tree. I have sent
_a few_ emails to them after completing this for-fun project regarding **GPL violations**, but I have yet to hear back nor
do I expect that I ever will. Because of these hindrances, I had to get a teensy bit creative in my process. This post
should cover most of my pains throughout my attempt, as well as offer guidance for anyone looking to hack their own
CaptionCall device (_Note: you can find tons of these things on eBay for about $25 each!_)

<!--description-->

## Looking for an Entrance
So far, all I knew is that based on the firmware build ID this thing I had in my hands ran Linux. That's really all
I could find looking at the surface. Additionally, after hooking it up to my network and running a port scan against
the device, nothing was open. Drat! Time to switch to plan b and open 'er up! Removing the case was fairly trivial:
3 philips-head screws were the only things securing the back in place; 2 on the bottom left and right, and one more
hidden below an adhesive rubber strip. After prying the two halves apart, two headers on the PCB immediately caught
my attention. The first looks like a standard ARM JTAG pad; however the leads from the traces to the SoC were cut.
Kind of a bummer. Not unfixable, but I really didn't feel like soldering tiny jumper wires and inevitably burning off
my fingertips. The second header definitely appeared familiar too. I guessed it was for UART access, and after probing
it with a multimeter my assumption turned out to be correct!

![The UART header in all its glory](https://i.imgur.com/xJSJ0vI.jpeg)

So, what can I do with UART on a CaptionCall, you ask? Apparently access the system console! **However**, there was one
tiny catch. After completing booting, I was greeted with a friendly login prompt:

```
U-Boot 2009.08 (Aug 31 2016 - 17:49:09)

CPU: Freescale i.MX6 family TO0.1 at 792 MHz

... REMOVED ...

Bluetooth: RFCOMM TTY layer initialized
Bluetooth: RFCOMM socket layer initialized
Bluetooth: RFCOMM ver 1.11
Bluetooth: BNEP (Ethernet Emulation) ver 1.3
Bluetooth: BNEP filters: protocol multicast
Bluetooth: HIDP (Human Interface Emulation) ver 1.2
lib80211: common routines for IEEE802.11 drivers
Registering the dns_resolver key type
VFP support v0.3: implementor 41 architecture 3 part 30 variant 9 rev 4

bonanza login:
```

I tried root/root, root/toor, etc. but none of them worked. I was stuck until I could dump `/etc/passwd` and crack
the hashed login password. But to do that, I'd need to access the ramdisk! And to access the ramdisk, I need a shell!
I was stuck in a catch-22 situation. All hope was not lost, however, because pressing 'Escape' within 1 second of
turning it on dropped me to a u-boot console! Still, the hardest part was yet to come...

```
u-boot> 
```

## Dumping the kernel
U-boot comes with a few helpful utilities to make our life easier. This build let me play around with the loader
environment variables like the kernel cmdline. Initially I tried to change it to boot single-user and with a custom
ramdisk, but alas the kernel they built had built-in boot arguments and ignored those from u-boot. After a bit more
digging I discovered [this lovely article](https://cybergibbons.com/hardware-hacking/recovering-firmware-through-u-boot/)
on how to recover firmware through u-boot over serial (since mine lacked the `tftp` command).

Basically, what I did was load the first kernel partition (since for some reason this has an A/B layout) to a known
memory offset, and loop through each address value using `md.b` to print it over serial. This has the upside of being
relatively simple to run, but the downsides of (a) being error-prone due to physical interference on the UART, and (b)
being very, _very_, _VERY_ slow. It took about 3 hours to dump the entire kernel partition over 115200 baud. And to
make matters even worse, sections of the dump were corrupted due to the issue from (a)! That meant I had to make additional
dumps over serial and _merge_ them together in order to correct them! The whole process took about 9 hours, so I wrote a
script called `uboot_hexdump_parser.py` to do all the dirty work + error correction for me while I slept.

After waking up the following morning I was greeted with my 32MB, error-free kernel partition dump! Running `binwalk` told
me that I had a compressed Linux kernel neatly tacked onto a u-boot uImage header. Usually, you add the kernel and
ramdisk separately to the uImage to make life easier, but CaptionCall decided to go the painful route and embed the ramdisk
directly within the kernel. This caused a lot of suffering down the line, but I'll cover that later. Binwalk had no issue
stripping the xz-compressed ramdisk from the kernel and extracting it for me, huzzah! Poking around the files in `/etc`
I learned the partition layout:

```
# partition table of /dev/mmcblk0
unit: sectors

#---- U-Boot: 16 blocks for partition table -------------------------------------------------------
#/dev/mmcblk0   : start=0,     size=16               <---Primary partition table [dummy line]
 /dev/mmcblk0p1 : start=16,    size=4080,    Id=DA

#---- U-Boot env ----------------------------------------------------------------------------------
 /dev/mmcblk0p2 : start=4096,  size=512,     Id=DA

#---- Kernel 1 ------------------------------------------------------------------------------------
 /dev/mmcblk0p3 : start=4608,  size=65536,   Id=F0

#---- extended container --------------------------------------------------------------------------
 /dev/mmcblk0p4 : start=70144, size=7663100, Id=85

#---- Kernel 2 ------------------------------------------------------------------------------------
 /dev/mmcblk0p5 : start=70656, size=65536,   Id=F0

#---- Data ----------------------------------------------------------------------------------------
 /dev/mmcblk0p6 : start=136192, size=7597052, Id=83
```

Everything is run from the ramdisk and most of that 4GB EMMC I was talking about is mounted to `/mnt/data` and contains
user-stored contact info, help documents, images, recordings, etc. rather than a rootfs like I initially assumed. This meant
if I wanted to change anything or stop the phone UI from loading on boot, I'd have to change the ramdisk. Nonetheless,
I had gotten this far and I wanted to know what the default root password was. Running `john` took about 8 seconds
to tell me the original unhashed root password was `y4u8it`. I tried that and was greeted with a busybox shell. So,
I told myself, I'm definitely one step closer to getting DOOM working. :)

## Modifying the Ramdisk
CaptionCall's proprietary UI started on boot and hogged the framebuffer, so I quickly realized it had to go. The problem
was that it was added in `/etc/inittab` to automatically restart when killed. And since the ramdisk was read-only I couldn't
just comment it out from my root shell. Plus, even if I did manage to remount `/` as read-write, it wouldn't persist past
a reboot! I had to figure out how to change the ramdisk at-will.

This took me on an exciting adventure in learning how kernel decompression works under the hood. You see, there are two general
types of kernel images: "uncompressed" images (often named Image or vmlinux) which are your standard, run-of-the-mill kernels,
and "compressed" images (often named \*zImage or vmlinuz). For the compressed images, decompressor code (called `piggy`) is
added at the beginning of the file, and the rest of the kernel data is compressed using gzip, bzip2, xz, etc. It looks a bit
like this:

```
-----------------
| uImage header |
-----------------
|    piggy.o    |
-----------------
|               |
|               |
| Gzip'd kernel |
|               |
|  -----------  |
|  | Ramdisk |  |
|  -----------  |
|               |
-----------------
```

As it also turns out, you can just remove the decompressor code, extract the kernel data following it, and it will work
just fine as if it was still compressed! That's one less layer of encapsulation to worry about. Modying the ramdisk **inside**
the kernel image is a bit trickier. Linux provides two ELF symbols, `__irf_start` and `__irf_end` to mark the beginning and end
of the embedded ramdisk, respectively. However, my kernel image wasn't in ELF format. Fortune favored me again, and I found
a neat utility called [vmlinux-to-elf](https://github.com/marin-m/vmlinux-to-elf) that converts all manner of mutilated kernel
blobs to ELF format. With this I was able to grep the ramdisk boundary offsets and begin patching! In the end, I wrote a second
script called `uImage_rdpatch.sh` that deals with everything from modifying the ramdisk to even patching the touchscreen
driver (which, as I found through way too much trial-and-error, had an evdev bug that prevented it from working under X).

My new ramdisk cleans up the `inittab` and also automatically runs any script called `/mnt/data/init.sh` on boot! Perfect for
my next evil plan, which was creating a Debian chroot to auto-start X11, IceWM, and DOOM...

## The Glorious Debian Chroot!
Since I really didn't want to hassle with cross-compiling DOOM for an embedded buildroot ramdisk running an ancient version of
libc, I opted to use debootstrap and create an armel `jessie` chroot, which, unfortunately, is the last version of ARM Debian
to support glibc 2.23 and the CaptionCall Linux 3.0.35 kernel version. Luckily, this part went without much hassle
(other than debugging the extremely strange touchscreen issues I ended up having to patch in the kernel, hmph)! Using my l33t
Xorg.conf trial-and-error skills, I eventually wrangled fbdev, mesa, and evdev to load IceWM on the screen with touchscreen and
keypad support. A bit more hacking and audio, bluetooth, and finally GPIO support was complete within my chroot.

Then finally, the moment of truth...

It can run DOOM!!!

![Behold, DOOM!](https://media3.giphy.com/media/O3pJDxrpzSJMun8Ky0/giphy.gif)

## The Code
If you'd like to perform this hack yourself, I provide a pre-patched kernel, as well as all of the utilities listed in this
post over at my GitHub repository [here](https://github.com/joshumax/dumping-ground/tree/master/bonanza_hacks). All that's
required is for you to open the phone and connect a UART dongle to the plastic header, login using `root:y4u8it`, and finally
write the patched kernel to NAND by typing `dd if=kernel1.patched of=/dev/mmcblk0p3`. Have fun and happy hacking!
