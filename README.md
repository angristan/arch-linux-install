# Arch Linux installation guide

## Introduction

These are my notes on installing [Arch Linux](https://www.archlinux.org/). This is not meant to be a universal guide, but only how I like to setup Arch Linux on my workstations. Since other people might find it useful, I decided to publish it.

Here is the setup I use:

- UEFI
- systemd-boot
- Encrypted disk + swap, plain `/boot`
- NetworkManager
- Xorg
- KDE / Plasma
- SDDM

This is mostly based on the [installation guide](https://wiki.archlinux.org/index.php/Installation_guide). I kept what I needed and added other parts. I made sure to put the links to all the wiki pages that I used. (❤️ Arch Wiki)

## Inital setup

If using a French keyboard:

```sh
loadkeys fr
```

Check if system is under UEFI:

```sh
ls /sys/firmware/efi/efivars
```

Connect to wifi if needed

```sh
wifi-menu
```

Enable NTP and set timezone

```sh
timedatectl set-ntp true
timedatectl set-timezone Europe/Paris
```

## Partitionning

- https://wiki.archlinux.org/index.php/Partitioning
- https://wiki.archlinux.org/index.php/EFI_system_partition

Here, a NVMe disk is used.

`cfdisk` is my favorite partitionning ncurses tool.

```sh
cfdisk /dev/nvme0n1
```

Choose GPT if asked.

Partitions:

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | 512M   | EFI System       |
| /dev/sda2  | xG     | Linux Filesystem |
| /dev/sda3  | 4G     | swap             |

Create the partitions, then label them. Then write and quit.

## File systems and LUKS encryption

- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS

Prepare the encrypted container:

```sh
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/sda2
cryptsetup open /dev/sda2 cryptroot
```

Format and mount it:

```sh
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```

Format the boot partition and mount it:

```sh
mkfs.fat -F32 /dev/sda1
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

Create and enable swap:

```sh
mkswap /dev/sda3
swapon /dev/sda3
```

Check if it's working with:

```sh
free -h
```

## Install system

Install the base packages:

```sh
pacstrap /mnt base base-devel
```

## System setup

Generate partition table:

```sh
genfstab -U /mnt > /mnt/etc/fstab
```

Enter the chroot:

```sh
arch-chroot /mnt
```

Set timezone and sync clock:

```sh
ln -sf /usr/share/zoneinfo/Europe/Paris /etc/localtime
hwclock --systohc
```

Uncomment `en_US.UTF-8 UTF-8` (and `fr_FR.UTF-8 UTF-8` if needed) in `/etc/locale.gen`.

Generate locales:

```sh
locale-gen
```

Set default locale:

- https://wiki.archlinux.org/index.php/Locale#LC_COLLATE:_collation

```sh
echo "LANG=en_US.UTF-8
LC_COLLATE=C" > /etc/locale.conf
```

Set keymap to French (if needed):

```sh
echo "KEYMAP=fr-latin9" > /etc/vconsole.conf
```

Set hostname:

```sh
echo "arch" > /etc/hostname
```

Set hosts file:

```sh
echo "127.0.0.1 localhost
::1 localhost
127.0.1.1 arch.localdomain arch" >> /etc/hosts
```

Set root password:

```sh
passwd
```

## Inital ramdisk

- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio
- https://wiki.archlinux.org/index.php/Mkinitcpio

Update the following line in `/etc/kminitcpio.conf`:

```sh
HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)
```

Generate the ramdisks using the presets:

```sh
mkinitcpio -P
```

## Setup systemd-boot as the bootloader

- https://wiki.archlinux.org/index.php/Systemd-boot
- https://bbs.archlinux.org/viewtopic.php?id=230911

Since we're using `/boot/`, no need to use `--path` option.

```sh
bootctl install
```

`/boot/loader/entries/arch.conf` :

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options rd.luks.name=<sda2 UUID>=cryptroot root=UUID=<dm-0> rw
```

Note: `root` can also be `root=/dev/mapper/rootcrypt`.

To get the [partition UUID](https://wiki.archlinux.org/index.php/Persistent_block_device_naming#by-uuid) easily (since we can't copy/paste anything at this point):

```sh
ls -l /dev/disk/by-uuid/ | grep sda2 | cut -f 10 -d " " >> /boot/loader/entries/arch.conf
ls -l /dev/disk/by-uuid/ | grep dm-0 | cut -f 10 -d " " >> /boot/loader/entries/arch.conf
```

## Encrypted swap

- https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption#UUID_and_LABEL

This setup will use `crypttab` to initalize a swap parition on `/dev/sda2`, encrypted with a key from `/dev/urandom`, upon each boot.

Thus, when the machine is shutdown, the key is lost and the content of the swap partition can't be read.

As usual, we want to use UUID since they are safer to use than their `/dev` mappings. Also, the partition will be wiped upon each reboot so it's very important to make sure the same partition is always used.

Since it will be wiped, we can't use a UUID or a label to identify it. Except if we create a tiny, empty file system with a label and then define an offset in `crypttab`.

```sh
mkfs.ext2 -L cryptswap /dev/sda3 1M
```

`/etc/crypttab`:

```
swap LABEL=cryptswap /dev/urandom swap,offset=2048,cipher=aes-xts-plain64,size=512
```

`/etc/fstab`:

```
/dev/mapper/swap none swap defaults 0 0
```

## Intel Microcode

- https://wiki.archlinux.org/index.php/Microcode 
if Ryzen or amd cpu amd-ucode
- https://www.archlinux.org/packages/core/any/amd-ucode/

```sh
pacman -S intel-ucode
```
(Same here for amd-ucode)
Add `initrd /intel-ucode.img` above `initrd /initramfs-linux.img` in `/boot/loader/entries/arch.conf`.

Check after reboot:

```sh
dmesg -T | grep microcode
```

## Networking with NetworkManager

- https://wiki.archlinux.org/index.php/NetworkManager

```sh
pacman -S networkmanager
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager
```

If wired with DHCP, nothing more to do.

## Reboot! (if you want)

As this point, the system should be working and usable, you we can reboot. You can also stay in the chroot.

```sh
exit
umount -R /mnt
cryptsetup close cryptroot
reboot
```

## User account

To enable sudo access, uncomment this line in `/etc/sudoers`:

```
%wheel ALL=(ALL) ALL
```

The `sudo` group does not exit by default so we'll use `wheel`.

Add user:

```sh
useradd -m -g wheel -c 'Stanislas' -s /bin/bash stanislas
passwd stanislas
```

## Xorg

- https://wiki.archlinux.org/index.php/Xorg

```sh
pacman -S xorg-server
pacman -S xf86-video-intel
```

## Desktop environment: Plasma and KDE

- https://wiki.archlinux.org/index.php/KDE

```sh
pacman -S plasma
pacman -S kde-applications
```

## Display manager: SDDM

- https://wiki.archlinux.org/index.php/SDDM

```sh
sudo pacman -S sddm
sudo systemctl enable sddm
sudo systemctl start sddm
```

By default, Arch uses the `archlinux-simplyblack` theme, which is ugly. Let's setup `breeze` (the *original* default):

```sh
mkdir /etc/sddm.conf.d/
echo "[Theme]
Current=breeze" > /etc/sddm.conf.d/theme.conf
```

Log back in.

## Fonts

```sh
pacman -S ttf-{bitstream-vera,liberation,freefont,dejavu} freetype2
```

## VMware

As I trained in a VMware Fusion VM, here are some notes.

- https://wiki.archlinux.org/index.php/VMware/Installing_Arch_as_a_guest

Install `xf86-video-vmware` instead of `xf86-video-intel`. `xf86-input-vmmouse` too.

To install the VMware tools:

```sh
pacman -S openvpn-vm-tools
sudo systemctl enable vmtoolsd
sudo systemctl start vmtoolsd
```

## Now what?

I recommand add this to network manager for disabling check internet on archlinux.org

echo "[connectivity]
uri=
interval=0" >> /etc/NetworkManager/NetworkManager.conf

See https://wiki.archlinux.org/index.php/General_recommendations
