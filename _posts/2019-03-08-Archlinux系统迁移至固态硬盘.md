---
layout: post
title: "Archlinux系统迁移至固态硬盘"
author: "Shoto"
tag: "linux"
data:
categories: "记录"
---

给台式电脑配了块240G的固态硬盘。不想重做系统，所以准备直接把原来的 archlinux 系统迁移到固态硬盘上。 网上找了一下大要的方法，一类是用 dd，一类是用 rsync 的方式。对dd不熟，所以选了 rsync 方式。
找到的参考文档在这里[：archlinux 系统迁移](https://www.sdvcrx.com/post/2014-10-29-migration-archlinux-from-hdd-to-ssd/)。稍微有点旧，里面也还有一个错误。一会说到。

## 1. 分区

先进行分区，EFI方式的话，一定要分一个 fat32 的 256-550M大小的分区出。我的分区方案大概是这样的：

原有机械硬盘分区：

```
/dev/sda1         以前有 windows 里的 efi 分区，但老硬盘有问题，实际没有用
/dev/sda2         /boot
/dev/sda3         /
/dev/sda4         swap
```
新固态硬盘的分区：

```
/dev/sdb1       准备的efi分区
/dev/sdb2       /
/dev/sdb3       swap
```
分区要注意的是，对一分出来准备用的ESP分区，一定要注意查看一下是不是有 EFI System 的标意。我迁移时中间引导不起来，后来现是在 fdisk 区时，没有写巻标，所以认不出来是ESP区。

## 2. 迁移

接下来就是磁盘的挂迁移，需要一个 linux 系统的 livecd ，然最好是 arch 。刚开始时我相偷懒，觉得既然已经有一个系统了，直接在系统里挂固态硬盘，但同步到一半的时候会卡住,livecd 里不会。

而且如果直接在当前系统里挂硬盘同步，还需要特别注意在同步时排除掉被挂载的磁盘文件目录，要不然会陷入死循环，直至磁盘空间被占用完。我第一次就是把固态磁盘挂在/mnt/ssd 下面，但在同步时没有从源目录中排除掉 /mnt/ssd，造成了 ssd的mnt中又同步了 ssd 的死循环。

方法基本是建目录，挂载磁盘，然后同步。

```
# mkdir /mnt/ssd
# mkdir /mnt/hdd
# mount /dev/sda3 /mnt/hdd
# mount /dev/sdb2 /mnt/ssd
# mkdir /mnt/ssd/boot
# mkdir /mnt/ssd/boot/efi
# mount /dev/sda2 /mnt/hdd/boot
# mount /dev/sdb1 /mnt/ssd/boot/efi
```
archlinux的wiki上写的，好像 EFI分区可以直接挂成 /boot 或 /EFI，但我没试成功，还是挂在了 /boot/efi 下面。 这块后来也出了好大的一块问题。

然后是同步，我240G的文件，大概用了一个小时左右。

```
# rsync -aAXHv /mnt/hdd/ /mnt/ssd/
```
这里上文提到参考教程里有点小错误，目标路径不正确。目标路径是你的目标盘的根目录。

## 3. 重装引导

同步完就可以来重装引导了，在这里之前没搞懂EFI问题，所以总是不对，后来用 livecd 下，chroot 进系统，直接安装了一下 grub，完成了安装。

先是需要chroot去，在 chroot 之前，还需要mount几别的

```
# mount -t proc none /mnt/ssd/proc
# mount -o bind /dev /mnt/ssd/dev
# mount -t /sys   /mnt/ssd/sys
# mount -t /run  /mnt/ssd/run
```
然后chroot进系统

```
# chroot /mnt/ssd /bin/bash
```
进入系统后安装grub并设置。

```
# pacman -S grub-efi-x86_64
# pacman -S efibootmgr
# pacman -S os-prober
```
配置grub
```
# grub-install --target=x86-64-efi --efi-directory=/boot/efi --bootloader-id=archlinux
# grub-mkconfig -o /boot/grub/grub.cfg
```
到这里还有关键的一步，需要安装 rEFInd。 

```
# pacman -S refind-efi
# refind-install  --usedefault /dev/sdb1
# mkrlconf
```
sdb1是我磁盘上EFI分区的位置。到这里应该就可以了。

好多教程上没有安装 rEFInd，我到这里卡了好久，安装完后总是引导不成功，提示系统不支持EFI。有可能是我原来的系统不是efi方式，所以内核不支持，在这里升级下内核也有可能能解决这个问题。我是升内核和装 rEFInd 都做了，所以不知道是哪一个起作用了。

## 4. 修改fstab

接下来就是到 /etc/fstab 里面把你的磁盘和 UUID 修改一下。

```
# nano /mnt/ssd/fstab
```
注意里面内容写对，我写错了一字母，导致引导的时候加互不上 /efi，系统能进命令行，在命令行下挂载 efi 分区后，输 exit 就可以进入X界机。所以这里还是要注意

因为是 rsync 来的，所以原来的 fstab 里面分区和UUID有可能是不正确的，按你的实际情况来修改就行了。

查看 UUID 可以用
```
lsblk -f
```

### 5. 重启完成

然后就是重新启动，恭喜你，系统已经成功迁移到固态硬盘了。