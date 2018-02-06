---
layout: post
title: "Installation of centos on fake raid machine (intel RST)"
---

I try to install Unix system on a machine with fake raid supported by intel RST in BIOS.
But I spent 36 hours to install ubuntu 16.04, it did not come to succeed.
Accidentally, I saw a massage that installation of centos on fake raid is easier than ubuntu,
so I try to install and succeed. The reference website is [PowerRC](https://www.powerrc.net/intel-raid-fakeraid-centos.html).

Simply record the steps:

1. Build the raid in the BIOS. (On my X99 board, it is to set controller in RAID mode and restart.
Then find the Intel RST in advanced page. Follow the lead and construct a raid 5 array.)
And enable the CSM (compatibility supportive modules) option.

1. I use centos 7 disc to boot in UEFI mode. (Intel RST only support the UEFI BIOS for X99 chipset.)
Using the manual partition to build /boot (about 500M), /boot/efi (about 500M), swap and / and starting installation.

1. Reboot and use centos!

