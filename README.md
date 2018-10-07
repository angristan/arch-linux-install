# Arch Linux installation guide

## Introduction

These are my notes on installing [Arch Linux](https://www.archlinux.org/). This is not meant to be a universal guide, but only how I like to setup Arch Linux on my workstations. Since other people might find it useful, I decided to publish it.

Here is the setup I use:

- UEFI
- NVMe disk
- systemd-boot
- NetworkManager
- Xorg
- KDE / Plasma
- SDDM

## Todo

- Disk encryption (cryptsetup, LUKS, LVM...)

## Resources

I've followed the *official* installation guide.

https://wiki.archlinux.org/index.php/Installation_guide

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

https://wiki.archlinux.org/index.php/Partitioning

https://wiki.archlinux.org/index.php/EFI_system_partition

Here, a NVMe disk is used.

`cfdisk` is my favorite partitionning ncurses tool.

```sh
cfdisk /dev/nvme0n1
```

Choose GPT if asked.

Partitions:

| Partition      | Space     | Type             |
|----------------|-----------|------------------|
| /dev/nvme0n1p1 | 512M      | EFI System       |
| /dev/nvme0n1p2 | 4G        | swap             |
| /dev/nvme0n1p3 | Remaining | Linux Filesystem |

Create the partitions, then label them. Then write and quit.

Format partitions:

```sh
mkfs.fat -F32 /dev/nvme0n1p1
mkfs.ext4 /dev/nvme0n1p3
```

Create and enable swap:

```sh
mkswap /dev/nvme0n1p2
swapon /dev/nvme0n1p2
```

Check if it's working with:

```sh
free -h
```

## Install system

Mount the partitions:

```sh
mount /dev/nvme0n1p3 /mnt
mkdir -p /mnt/boot
mount /dev/nvme0n1p1 /mnt/boot
```

Install base system

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

Uncomment `en_US.UTF-8 UTF-8` and `fr_FR.UTF-8 UTF-8` in `/etc/locale.gen`.

Generate locales:

```sh
locale-gen
```

Set default locale:

```sh
echo "LANG=en_US.UTF-8" > /etc/locale.conf
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

## Setup systemd-boot as the bootloader

https://wiki.archlinux.org/index.php/Systemd-boot

Since we're using `/boot/`, no need to use `--path` option.

```sh
bootctl install
```

`/boot/loader/entries/arch.conf` :

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=PARTUUID=xxx rootfstype=ext4 add_efi_memmap rw
```

To get the UUID easily (since we can't copy/paste anything at this point):

```
ls -l /dev/disk/by-partuuid/ | grep nvme0n1p3 | cut -f 10 -d " " >> /boot/loader/entries/arch.conf
```

`nvme0n1p3` beeing our `/` partition.

## Reboot! (if you want)

As this point, the system should be working and usable, you we can reboot. You can also stay in the chroot.

## Intel Microcode

https://wiki.archlinux.org/index.php/Microcode

```sh
pacman -S intel-ucode
```

Add `initrd /intel-ucode.img` above `initrd /initramfs-linux.img` in `/boot/loader/entries/arch.conf`.

Check after reboot:

```sh
dmesg -T | grep microcode
```

## Networking with NetworkManager

https://wiki.archlinux.org/index.php/NetworkManager

```sh
sudo systemctl enable NetworkManager
sudo systemctl start NetworkManager
```

If wired with DHCP, nothing more to do.

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

https://wiki.archlinux.org/index.php/Xorg

```sh
pacman -S xorg-server
pacman -S xf86-video-intel
```

## Desktop environment: Plasma and KDE

https://wiki.archlinux.org/index.php/KDE

```sh
pacman -S plasma
pacman -S kde-applications
```

## Display manager: SDDM

https://wiki.archlinux.org/index.php/SDDM

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

https://wiki.archlinux.org/index.php/VMware/Installing_Arch_as_a_guest

Install `xf86-video-vmware` instead of `xf86-video-intel`. `xf86-input-vmmouse` too.

To install the VMware tools:

```sh
pacman -S openvpn-vm-tools
sudo systemctl enable vmtoolsd
sudo systemctl start vmtoolsd
```

## Now what?

See https://wiki.archlinux.org/index.php/General_recommendations
