# Arch Linux installation guide

## Introduction

These are my notes on installing [Arch Linux](https://www.archlinux.org/). This is not meant to be a universal guide, but only how I like to setup Arch Linux on my workstations. Since other people might find it useful, I decided to publish it.

Here is the setup I use:

- UEFI
- systemd-boot
- LVM on LUKS, plain `/boot`
- NetworkManager
- Xorg
- KDE / Plasma
- SDDM

This is mostly based on the [installation guide](https://wiki.archlinux.org/index.php/Installation_guide). I kept what I needed and added other parts. I made sure to put the links to all the wiki pages that I used. (❤️ Arch Wiki)

## Table of content

- [Inital setup](#inital-setup)
- [Disk management](#disk-management)
  - [Method 1 - "Classic" (unencrypted)](#method-1---classic-unencrypted)
    - [Partitions](#partitions)
    - [File systems](#file-systems)
  - [Method 2 - LVM (unecrypted)](#method-2---lvm-unecrypted)
    - [Partitions](#partitions)
    - [LVM](#lvm)
    - [File systems](#file-systems)
  - [Method 3 - LUKS + crypttab](#method-3---luks--crypttab)
    - [Partitions](#partitions)
    - [LUKS](#luks)
    - [File systems](#file-systems)
    - [Encrypted swap](#encrypted-swap)
  - [Method 4 - LVM on LUKS](#method-4---lvm-on-luks)
    - [Partitions](#partitions)
    - [LUKS](#luks)
    - [LVM](#lvm)
    - [File systems](#file-systems)
- [Install system](#install-system)
- [System setup](#system-setup)
- [Initial ramdisk](#initial-ramdisk)
- [Bootloader: systemd-boot](#bootloader-systemd-boot)
- [Intel Microcode](#intel-microcode)
- [Networking: NetworkManager](#networking-networkmanager)
- [Reboot! (if you want)](#reboot-if-you-want)
- [User account](#user-account)
- [Xorg](#xorg)
- [Desktop environment: Plasma and KDE](#desktop-environment-plasma-and-kde)
- [Display manager: SDDM](#display-manager-sddm)
- [Fonts](#fonts)
- [VMware](#vmware)
- [Now what?](#now-what)

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

## Disk management

- https://wiki.archlinux.org/index.php/Partitioning
- https://wiki.archlinux.org/index.php/EFI_system_partition

I will describe below 4 ways of using your disk:

1. Classic, unencrypted partitions
2. LVM
3. LUKS + crypttab for the swap
4. LVM on LUKS

Most people use the 4th one these days.

`cfdisk` is my favorite partitionning ncurses tool.

For each method, you can launch the tool with:

```sh
cfdisk /dev/sda1
```

Choose GPT if asked. Create the partitions and label them. Then write and quit.

### Method 1 - "Classic" (unencrypted)

#### Partitions

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | 512M   | EFI System       |
| /dev/sda2  | xG     | Linux Filesystem |
| /dev/sda3  | xG     | swap             |

#### File systems

`/` partition:

```sh
mkfs.ext4 /dev/sda2
mount /dev/sda2 /mnt
```

`/boot` partition:

```sh
mkfs.fat -F32 /dev/sda1
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

`swap`:

```sh
mkswap /dev/sda3
swapon /dev/sda3
```

### Method 2 - LVM (unecrypted)

- https://wiki.archlinux.org/index.php/LVM

#### Partitions

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | 512M   | EFI System       |
| /dev/sda2  | xG     | Linux Filesystem |


#### LVM

Create the physical volume:

```sh
pvcreate /dev/sda2
```

Then the volume group:

```sh
vgcreate vg0 /dev/sda2
```

Then the logical volumes:

```sh
lvcreate -L xG vg0 -n swap
lvcreate -l 100%FREE vg0 -n root
```

#### File systems

`/` partition:

```sh
mkfs.ext4 /dev/vg0/root
mount /dev/vg0/root /mnt
```

`/boot` partition:

```sh
mkfs.fat -F32 /dev/sda1
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

`swap`:

```sh
mkswap /dev/vg0/swap
swapon /dev/vg0/swap
```

### Method 3 - LUKS + crypttab

#### Partitions

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | 512M   | EFI System       |
| /dev/sda2  | xG     | Linux Filesystem |
| /dev/sda3  | xG     | swap             |

#### LUKS

- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Simple_partition_layout_with_LUKS

Prepare the encrypted container:

```sh
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/sda2
cryptsetup open /dev/sda2 cryptroot
```

#### File systems

`/`:

```sh
mkfs.ext4 /dev/mapper/cryptroot
mount /dev/mapper/cryptroot /mnt
```

`/boot`:

```sh
mkfs.fat -F32 /dev/sda1
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

#### Encrypted swap

- https://wiki.archlinux.org/index.php/Dm-crypt/Swap_encryption#UUID_and_LABEL

This setup will use `crypttab` to initalize a swap parition on `/dev/sda2`, encrypted with a key from `/dev/urandom`, upon each boot.

Thus, when the machine is shutdown, the key is lost and the content of the swap partition can't be read.

As usual, we want to use UUID since they are safer to use than their `/dev` mappings. Also, the partition will be wiped upon each reboot so it's very important to make sure the same partition is always used.

Since it will be wiped, we can't use a UUID or a label to identify it. Except if we create a tiny, empty file system with a label and then define an offset in `crypttab`.

After entering pacstraping the system and entering `arch-root`:

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

### Method 4 - LVM on LUKS

- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS

#### Partitions

| Partition  | Space  | Type             |
|------------|--------|------------------|
| /dev/sda1  | 512M   | EFI System       |
| /dev/sda2  | xG     | Linux Filesystem |

#### LUKS

```sh
cryptsetup luksFormat --type luks2 --cipher aes-xts-plain64 --key-size 512 /dev/sda2
cryptsetup open /dev/sda2 cryptlvm
```

#### LVM

Create the physical volume:

```sh
pvcreate /dev/mapper/cryptlvm
```

Then the volume group:

```sh
vgcreate vg0 /dev/mapper/cryptlvm
```

Then the logical volumes:

```sh
lvcreate -L xG vg0 -n swap
lvcreate -l 100%FREE vg0 -n root
```

#### File systems

`/` partition:

```sh
mkfs.ext4 /dev/vg0/root
mount /dev/vg0/root /mnt
```

`/boot` partition:

```sh
mkfs.fat -F32 /dev/sda1
mkdir /mnt/boot
mount /dev/sda1 /mnt/boot
```

`swap`:

```sh
mkswap /dev/vg0/swap
swapon /dev/vg0/swap
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

See https://wiki.archlinux.org/index.php/Locale

Uncomment `en_US.UTF-8 UTF-8` (and `fr_FR.UTF-8 UTF-8` if needed) in `/etc/locale.gen`.

Generate locales:

```sh
locale-gen
```

Set default locale:

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

## Initial ramdisk

- https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio
- https://wiki.archlinux.org/index.php/Mkinitcpio

The `HOOKS` line might need to be updated in `/etc/kminitcpio.conf` depending on the disk method you used:

- Method 1: nothing to change
- Method 2: `base systemd udev autodetect modconf block sd-lvm2 filesystems keyboard fsck`
- Method 3: `base systemd udev autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck`
- Method 4: `base systemd udev autodetect keyboard sd-vconsole modconf block sd-encrypt sd-lvm2 filesystems fsck`

Generate the ramdisks using the presets:

```sh
mkinitcpio -P
```

## Bootloader: systemd-boot

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
options ...
```

The `options` line depends on the disk method you used.

- Method 1: `options root=UUID=<sda2 UUID> rw`
- Method 2: `options root=/dev/vg0/root rw`
- Method 3: `options rd.luks.name=<sda2 UUID>=cryptroot root=/dev/mapper/cryptroot rw`
- Method 4: `options rd.luks.name=<sda2 UUID>=cryptlvm root=/dev/vg0/root rw`

To get the [partition UUID](https://wiki.archlinux.org/index.php/Persistent_block_device_naming#by-uuid) easily (since we can't copy/paste anything at this point):

```sh
ls -l /dev/disk/by-uuid/ | grep <partition name> | cut -f 9 -d " " >> /boot/loader/entries/arch.conf
```

## Intel Microcode

- https://wiki.archlinux.org/index.php/Microcode

```sh
pacman -S intel-ucode
```

Add `initrd /intel-ucode.img` above `initrd /initramfs-linux.img` in `/boot/loader/entries/arch.conf`.

Check after reboot:

```sh
dmesg -T | grep microcode
```

Use `amd-ucode` for an AMD CPU.

## Networking: NetworkManager

- https://wiki.archlinux.org/index.php/NetworkManager

```sh
pacman -S networkmanager
systemctl enable NetworkManager
systemctl start NetworkManager
```

If wired with DHCP, nothing more to do.

## Reboot! (if you want)

As this point, the system should be working and usable, you we can reboot. You can also stay in the chroot.

```sh
exit
umount -R /mnt
# cryptsetup close {cryptroot,crptlvm}
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
pacman -S sddm
systemctl enable sddm
systemctl start sddm
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
systemctl enable vmtoolsd
systemctl start vmtoolsd
```

## Now what?

See https://wiki.archlinux.org/index.php/General_recommendations
