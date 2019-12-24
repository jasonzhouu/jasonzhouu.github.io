---
layout: post
title:  "将多个操作系统迁移到同一个硬盘"
date:   2019-12-21
tags:   [操作系统]
---

## 背景
我有2台笔记本电脑，一台旧的，一台相对新一些的；多块硬盘，分别安装了我不同时期使用的操作系统。
- 旧的：BIOS，用1块100GB机械硬盘安装了双系统: Debian 和 Manjaro;
- 新的：UEFI，有2 块硬盘，其中750GB机械硬盘安装了 Manjaro 系统，200GB固态硬盘安装了Win10 系统；

我最常用的系统是 Manjaro（安装在新电脑的750G硬盘上的那个），其次是Debian，再其次是 Win10。
可以看到，我最常用的Manjaro安装在相对低效的机械硬盘，最少用的Win10却安装在高效的固态硬盘。
于是我想将这几个系统都迁移到200GB固态硬盘上，以达到固态硬盘的充分利用。
在这迁移过程中牵涉到了UEFI、GRUB的知识，在学习之后，成功达到了这个目标。当之后整理了一下思路，想到了更高效的流程，现记录如下。

## 工具
- gnome disk：用于查看分区情况；
- gparted：用于将Debian和Manjaro所在分区缩小到尽量小，以缩短dd命令的执行时间，或腾出空间；
- dd 命令：用于拷贝分区；
- 硬盘盒：用于将硬盘的SATA接口转USB接口；

## 思路
将Debian和Manjaro系统的分区都拷贝到已经安装了Windows的硬盘，然后chroot到Manjaro分区，执行grub-install, update-grub命令。

## 具体步骤

### 1. 缩小Manjaro,Debian,Windows系统分区
启动旧电脑的Manjaro系统，将750GB机械硬盘、200GB固态硬盘插入硬盘盒，通过USB口连接到旧电脑。
打开gparted程序：
- 将Debian、Manjaro系统的分区缩至最小
- 将Windows系统的分区缩小到剩余10GB空白空间

### 2. 将Debian,Manjaro拷贝到固态硬盘
用gparted程序在固态硬盘的空白位置新建3个分区，分别用于放Debian、Manjaro、Swap分区，根据系统所需要的最小大小及使用频率设置分区空间。


使用dd命令将Debian分区拷贝到固态硬盘：
```
sudo dd if=/dev/sda* of=/dev/sdb bs=10MB status=progress
```
解释：`status=progress`表示拷贝过程显示速度和进度，`bs=10MB`以10MB为最小单位进行拷贝，默认是512bytes，也就是硬盘的一个sector的尺寸，但是实测这样速度较慢，可能是读写单位尺寸太小、读写频率太高，会降低读写速度的缘故。

之后用同样的命令将Manjaro分区拷贝到固态硬盘。

### 3. `chroot` 到Manjaro 分区，在ESP分区安装grub 程序，以及更新grub的配置文件[^1]
```
sudo mount /dev/sdXXX /mnt
sudo mount /dev/sdXX /mnt/boot/efi
for i in /dev /dev/pts /proc /sys /run; do sudo mount -B $i /mnt$i; done
sudo chroot /mnt
grub-install /dev/sdX
update-grub 
```
解释：
- 电脑启动时，需要boot loader加载操作系统，Linux一般使用grub作为 boot loader。
- 电脑启动后时，首先UEFI硬件会加载ESP(EFI system partition) 分区内的文件，`grub-install`命令就是在这个分区内存入grub程序；
- ESP分区内的grub程序被UEFI加载后，会再加载指定操作系统分区的的normal 模块，`grub-install`时会将操作系统分区的位置硬编码到grub程序，或者生成配置文件放在ESP分区。normal的路径通常是`/boot/grub/x86_64-efi/normal.mod`；
- normal模块加载其他模块，以及配置文件`/boot/grub/grub.cfg`，并会生成类似下图的界面[^3]，让用户选择接下来要启动的系统。[^2]列表项、背景、文字颜色等等都是在这个配置文件内指定，不过这个配置文件一般是由`update-grub`命令生成，而不直接对其进行编辑。`update-grub`会自动检查当前有哪些系统分区，生成这个操作系统列表。`update-grub`会加载`/etc/default/grub`, `/etc/grub`配置文件[^4][^5]，这个就是用户需要编辑的配置文件。我只需要`update-grub`生成操作系统列表就够了，不需要更改配置文件。

![grub menu interface](https://forum.manjaro.org/uploads/default/optimized/3X/0/e/0e33a9622f1b71643577695f4836e5ebdefcb1fc_2_668x500.png)

另外：
- 检查`/etc/default/grub`中是否用到的UUID是否有错，如果有错，则先改正然后执行`update-grub`命令；
- 检查`/etc/fstab`是否有错。

### 4. 将固态硬盘安装到UEFI 的新电脑，启动，到此结束。


## 补充
### 1. BIOS, UEFI
在BIOS和UEFI这两种硬件下，启动的第一个阶段有很大不同，因此grub在这两种硬件下也有相应的不同：
- BIOS启动后，先加载MBR，所以`grub-install`在MBR安装`boot.img`，但MBR太小，只要512bytes，无法做太多事情，所以它只作为跳板，加载后面的程序。也就是MBR与第一个分区的间隙，这个空间也不大，通常只有32KB，但是足够访问`/boot/grub`路径，加载其他模块（用于生成 grub menu、加载操作系统等功能的模块），`grub-install`在这里存放文件被称为`core.img`。[^6]
- UEFI能够识别GPT分区表和FAT文件系统，启动后会会直接加载ESP分区，而不用像BIOS先从MBR跳转。UEFI从ESP分区加载其中的EFI 应用。`grub-install`在里面存放了grub程序，即`grubx86.efi`，它的作用和BIOS下的`core.img`作用一样，不过是按照EFI的接口进行编译的。[^7]如果只安装了一个操作系统，则不用加载grub，UEFI可以直接加载操作系统。

### 2. Win10
grub没法直接启动Win10，需要chainload，即传递给Win10自己的EFI程序。安装Win10后，在ESP分区可以看到有Microsoft目录，里面是Windows的bootloader。执行`update-grub`生成的配置文件会处理好这件事情，从而开机进入menu interface时，选择windows系统，会将权限传递给windows的程序。[^8]

查看`grub.cfg`配置文件的chainload部分，可以看到它加载了ESP分区下Microsoft目录的EFI程序：
```
menuentry 'Windows Boot Manager (on /dev/sda2)' --class windows --class os $menuentry_id_option 'osprober-efi-6DD8-DACA' {
        savedefault
        insmod part_gpt
        insmod fat
        set root='hd0,gpt2'
        if [ x$feature_platform_search_hint = xy ]; then
          search --no-floppy --fs-uuid --set=root --hint-ieee1275='ieee1275//disk@0,gpt2' --hint-bios=hd0,gpt2 --hint-efi=hd0,gpt2 --hin
t-baremetal=ahci0,gpt2  6DD8-DACA
        else
          search --no-floppy --fs-uuid --set=root 6DD8-DACA
        fi
        chainloader /efi/Microsoft/Boot/bootmgfw.efi
}
```

## 反思

相比较而言，2种做事方式：

- 定目标-》摸索-》遇到问题-》查文档-》再摸索-》总结

- 定目标-》查文档-》定方案-》摸索-》总结

后一种可能效率要高得多。

---

[^1]: [How can I reinstall GRUB to the EFI partition?](https://askubuntu.com/a/831241/798614)

[^2]: [How does GNU GRUB work](https://0xax.github.io/grub/)

[^3]: [https://www.gnu.org/software/grub/manual/grub/html_node/Menu-interface.html ](https://www.gnu.org/software/grub/manual/grub/html_node/Menu-interface.html)

[^4]: [Grub - Debian Wiki](https://wiki.debian.org/Grub)

[^5]: [GRUB 2 bootloader - Full tutorial](https://www.dedoimedo.com/computers/grub-2.html)

[^6]: https://www.gnu.org/software/grub/manual/grub/grub.html#Images

[^7]: [https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/s2-grub-whatis-booting-uefi](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/6/html/installation_guide/s2-grub-whatis-booting-uefi)

[^8]: https://www.gnu.org/software/grub/manual/grub/grub.html#DOS_002fWindows
