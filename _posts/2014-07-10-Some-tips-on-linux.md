---
layout: post
title: Some tips on Linux
---

This post is used to record some tips I can't categorize in ubuntu.

i. automatically load some shell scripts
In my system ubuntu 14.04, I can find the file `.bashrc` in my home directory.
Since I want ubuntu to load intel complier and mkl parameter automatically, all I need to do is to add the two lines in the end of that file: (mint 17: `gedit /etc/bash.bashrc`)

``` bash
source /opt/intel/composer_xe_2015/mkl/bin/mklvars.sh intel64
source /opt/intel/composer_xe_2015/bin/compilervars.sh intel64
```

Then I success!!

ii. cannot install ubuntu or Mint
With the options - acpi=off nolapic noapic, I finally install ubuntu successfully.

iii. cannot boot without nolapic, however, it only recognize one cpu with nolapic
I solved this problem by [Dual core recognized as single core because of nolapic?](http://ubuntuforums.org/showthread.php?t=1084622).
I edited the grub file with following commands:

``` bash
sudo bash
gedit /etc/default/grub
```

And replace `nolapic` with `pci=assign-busses apicmaintimer idle=poll reboot=cold,hard`, the grub file would be contain this two lines:

``` bash
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash acpi_osi=linux"
GRUB_CMDLINE_LINUX="noapic pci=assign-busses apicmaintimer idle=poll reboot=cold,hard"
```

Then use following command to update grub. And the problem is fixed.

``` bash
sudo update-grub
```

iv. to get the permission of ntfs disks, you can edit the fstab in /etc as following:

``` bash
sudo gedit /etc/fstab
```

And you can find the uuid by using the command `ls -l /dev/disk/by-uuid`. To add the disk and set the permission in the file fstab like this:

``` bash
UUID=1c712d26-7f9d-4efc-b796-65bee366c8aa / ext4    noatime,nodiratime,discard,errors=remount-ro 0       1
UUID=9298D0AB98D08EDB /media/Windows ntfs defaults,uid=1000,gid=1000,umask=002     0      0
UUID=08C2997EC29970A4 /media/Download ntfs defaults,uid=1000,gid=1000,umask=002      0      0
UUID=01CD524F3352C990 /media/Files ntfs defaults,uid=1000,gid=1000,umask=002      0      0
```

Then you can access your ntfs disk and set an alias for each disk.

v. use grub comstomer to edit the boot order. Installation:

``` bash
sudo add-apt-repository ppa:danielrichter2007/grub-customizer
sudo apt-get update
sudo apt-get install grub-customizer
```

vi. Install font `Inconsolata`[Download Here](http://www.levien.com/type/myfonts/inconsolata.html) and unity tweak tool (`sudo apt-get install unity-tweak-tool`).

vii. Install the chinese input `fcitx` and language `Chinese Traditional`.

``` bash
sudo apt-get install fcitx fcitx-chewing fcitx-config-gtk fcitx-frontend-all fcitx-module-cloudpinyin fcitx-ui-classic fcitx-frontend-qt4 fcitx-frontend-qt5 fcitx-frontend-gtk2 fcitx-frontend-gtk3
```

viii. Install ruby, jekyll and git.

``` bash
sudo apt-get install software-properties-common
sudo apt-add-repository ppa:brightbox/ruby-ng
sudo apt-get update
sudo apt-get install ruby2.2 ruby2.2-dev git python-pip python3-pip
sudo gem install jekyll
sudo pip install pygments
```

(To be continued.)
