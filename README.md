# Arch Linux with UEFI Install

## Bootable USB

### GNU/Linux

Run the following command, replacing `/dev/sdx` with your drive, e.g. `/dev/sdb`.

```
# dd bs=4M if=/path/to/archlinux.iso of=/dev/sdx status=progress && sync
```

### Windows

- Using [Rufus](https://rufus.akeo.ie/)

Simply select the Arch Linux ISO, the USB drive you want to create the bootable Arch Linux onto and click start.

- Using [USBwriter](https://sourceforge.net/projects/usbwriter/)

Just download the Arch Linux ISO, and with local administrator rights use the USBwriter utility to write to your USB flash memory

## Set the keyboard layout

```
# loadkeys br-abnt2
```

Available choices can be listed with ls `/usr/share/kbd/keymaps/**/*.map.gz`.

## Verify the boot mode

```
# ls /sys/firmware/efi/efivars
```

## Connect to the Internet

```
wifi-menu
```

Test it using `ping -c 3 www.google.com`.

## Update the system clock

```
# timedatectl set-ntp true
```

## Partition the disks

Identify the disk names with `lsblk` (results ending in rom, loop or airoot can be ignored).

In my case, I'm going to install Arch on `/dev/sda` and create two partitions:

|Partition|Type             |Size                |Description     |
|---------|-----------------|--------------------|----------------|
|sda1     |EFI System (ef00)|512mb               |Boot partition  |
|sda2     |Linux LVM (8e00) |Remaining free space|LVM partition   |

*See References below to find out the recommended swap size.*

### Create partitions

```
# gdisk /dev/sda
```

Type `o` to create a new empty GUID partition table (GPT).

Type `n` to create the boot partition.

```
Partition number: 1
First sector: Leave this blank
Size in sectors: +512M
Hex code or GUID: ef00  
```

Type `n` to create the LVM partition.

```
Partition number: 2
First sector: Leave this blank
Size in sectors: Leave this blank
Hex code or GUID: 8e00
```

Type `p` to print the partition table and confirm that it was created correctly.

Type `w` to write the partition table to disk and exit.

### Create LVM physical volume

```
# pvcreate /dev/sda2
```

Confirm that the physical volume was created correctly.

```
# pvdisplay
```

### Create LVM volume group

```
# vgcreate vg_linux /dev/sda2
```

Confirm that the volume group was created correctly.

```
# vgdisplay
```

### Create LVM logical volumes

I'm going to create three logical volumes:

|Logical Volume|Size |Description     |
|--------------|-----|----------------|
|lv_swap       |24gb |Swap partition  |
|lv_root       |20gb |Root partition  |
|lv_home       |200gb|Home partition  |

Create the swap partition.

```
# lvcreate -n lv_swap -L 24G -C y vg_linux
```

Create the root partition.

```
# lvcreate -n lv_root -L 20G vg_linux
```

Create the home partition.

```
# lvcreate -n lv_home -L 200G vg_linux
```

Confirm that the logical volumes were created correctly.

```
# lvdisplay
```

Our logical volumes should now be located in `/dev/mapper/` and `/dev/vg_linux`.
If you cannot find them, use the next commands to bring up the module for creating device nodes and to make volume groups available:

```
# modprobe dm_mod
# vgscan
# vgchange -ay
```

## Format the partitions

### EFI System Partition (ESP)

```
# mkfs.fat -F32 /dev/sda1
```

### Swap partition

```
# mkswap /dev/mapper/vg_linux-lv_swap
```

### Root partition

```
# mkfs.ext4 /dev/mapper/vg_linux-lv_root
```

### Home partition

```
# mkfs.ext4 /dev/mapper/vg_linux-lv_home
```

## Mount the partitions

### Root partition

```
# mount /dev/mapper/vg_linux-lv_root /mnt
```

### Home partition

```
# mkdir /mnt/home
# mount /dev/mapper/vg_linux-lv_home /mnt/home
```

### EFI System Partition (ESP)

```
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

Confirm that the file systems were mounted correctly.

```
# lsblk
```

## Rank the mirrors by speed

Back up the existing `/etc/pacman.d/mirrorlist`:

```
# cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Run the following sed line to uncomment every mirror:

```
# sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.backup
```

Finally, rank the mirrors. Operand `-n 6` means only output the 6 fastest mirrors:

```
# rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

## Install the base system

```
# pacstrap /mnt base base-devel
```

## Configure the system

### Generate `fstab` file

```
# genfstab -p /mnt > /mnt/etc/fstab
```

### Change root into the new system

```
# arch-chroot /mnt
```

### Set the time zone

```
# ln -s /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
```

### Run `hwclock` to generate `/etc/adjtime`

```
# hwclock --systohc --utc
```

### Locale

Edit the `/etc/locale.gen` file and uncomment the following locales

```
en_US.UTF-8 UTF-8
en_US ISO-8859-1
```

Generate the selected locales

```
# locale-gen
```

Add `LANG=your_locale` to `/etc/locale.conf`,

```
# echo "LANG=en_US.UTF-8" > /etc/locale.conf
```

Add console keymap and font preferences to `/etc/vconsole.conf`.

```
# echo "KEYMAP=br-abnt2" > /etc/vconsole.conf
```

### Hostname

Create an entry for your hostname in `/etc/hostname`.

```
# echo "myhostname" > /etc/hostname
```

It is recommended to also set the hostname in `/etc/hosts`:

```
#
# /etc/hosts: static lookup table for host names
#

#<ip-address>    <hostname.domain.org>    <hostname>
127.0.0.1        localhost.localdomain    localhost    myhostname
::1              localhost.localdomain    localhost    myhostname
```

### Enable multilib repository in `/etc/pacman.conf`

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

### Color output

Pacman has a color option. Uncomment the `Color` line in `/etc/pacman.conf`.

### Install basic software

```
# pacman -Syu

# pacman -S bash-completion iw wpa_supplicant dialog wireless_tools rfkill wpa_actiond ifplugd mlocate openssh
```

### Install Yaourt

Add the following repository in `/etc/pacman.conf`

```
[archlinuxfr]
SigLevel = Never
Server = http://repo.archlinux.fr/$arch
```

Update repository database and install Yaourt

```
# sudo pacman -Sy yaourt
```

### Automatic switching of network profiles

Find what your interfaces are called

```
# ip link
```

Package `ifplugd` for wired interfaces: After starting and enabling `netctl-ifplugd@interface.service` DHCP profiles are started/stopped when the network cable is plugged in and out.

```
# systemctl enable netctl-ifplugd@interface.service
```

Package `wpa_actiond` for wireless interfaces: After starting and enabling `netctl-auto@interface.service` profiles are started/stopped automatically as you move from the range of one network into the range of another network (roaming).

```
# systemctl enable netctl-auto@interface.service
```

### Install and configure sudo:

```
# pacman -S sudo
```

Run the `visudo` to edit the `/etc/sudoers` file

```
# EDITOR=nano visudo
```

Grant sudo access to users in the group wheel when enabled.

```
## Allows people in group wheel to run all commands
%wheel    ALL=(ALL)    ALL
```

### Activate NTP Client

```
# systemctl enable systemd-timesyncd
```

### Set root password

```
# passwd
```

### Add lvm2 hook to mkinitcpio.conf for root on LVM

Edit the file `/etc/mkinitcpio.conf` and insert `lvm2` between `block` and `filesystems` like so:

```
HOOKS="base udev ... block lvm2 filesystems ..."
```

### Create initramfs

```
# mkinitcpio -p linux
```

### Install bootloader

```
# bootctl install
```

### Create bootloader entry

Create a boot entry in `/boot/loader/entries/arch.conf`

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/mapper/vg_linux-lv_root rw
```

Create a default entry in `/boot/loader/loader.conf`

```
timeout 2
default arch
```

## Audio driver

```
# pacman -S pulseaudio pulseaudio-alsa lib32-libpulse lib32-alsa-plugins
```

## Install Xorg

```
# pacman -S xorg-xinit xorg-utils xorg-server xorg-xrandr xterm
```

## Video driver

### Intel

```
# pacman -S xf86-video-intel mesa mesa-libgl lib32-mesa-libgl vulkan-intel
```

### AMD

```
# pacman -S xf86-video-amdgpu mesa mesa-libgl lib32-mesa-libgl mesa-vdpau lib32-mesa-vdpau
```

### Nvidia

```
# pacman -S nvidia nvidia-libgl
# nvidia-xconfig
```

Check the list of attached graphic drivers

```
# startx
# xrandr --listproviders
```

## Touchpad

```
# pacman -S xf86-input-libinput
```

## Bluetooth

```
# pacman -S bluez bluez-utils

# systemctl enable bluetooth
```

## Media Transfer Protocol

```
# pacman -S libmtp gvfs-mtp
```

## Automounting removable media or network shares

```
# pacman -S autofs
```

## NTFS file system including read and write support

```
# pacman -S ntfs-3g
```

## Desktop Environment

### GNOME

Install GNOME desktop

```
# pacman -S gnome gnome-extra gnome-tweak-tool
```

Enable `gdm.service` to start GDM at boot time

```
# systemctl enable gdm.service
```

## Create user

```
# useradd -m -g users -G wheel -s /bin/bash myuser
# passwd myuser
```

## Finish installation

```
# exit
# umount -R /mnt
# reboot
```

## Switch to Network Manager

Install the network manager package and its GUI front-end.

```
# pacman -S networkmanager network-manager-applet
```

Disable *dhcpcd* and *netcl*, as network manager will replace both. So first let’s find our devices:

```
# ip link
```

Anything starting with *enp* is an ethernet device.

Anything starting with *wlp* is a wireless device.

Disable *dhcpcd* on any ethernet devices (my device was listed as `enp1s0`):

```
# systemctl disable dhcpcd@enp1s0.service
```

Disable *netctl* on any wireless devices  (my device was listed as `wlp2s0`):

```
# systemctl disable netctl-auto@wlp2s0.service
```

Enable network manager:

```
# systemctl enable NetworkManager.service
```

Finally, reboot.

```
# reboot
```

## Install optional software

### VirtualBox

Install the core packages.
```
# pacman -S virtualbox virtualbox-host-modules-arch
```

Load the VirtualBox kernel modules.
```
# modprobe vboxdrv boxnetadp vboxnetflt vboxpci
```

Add users that will be authorized to access host USB devices in guest to the vboxusers group.
```
# usermod -a -G vboxusers <login>
```

### Printing Service

```
# pacman -S cups cups-pdf gtk3-print-backends system-config-printer
# systemctl enable org.cups.cupsd.service
```

### HP Printers

```
# pacman -S hplip
```

## References

- [Arch Linux Installation guide](https://wiki.archlinux.org/index.php/installation_guide#Pre-installation)
- [USB flash installation media](https://wiki.archlinux.org/index.php/USB_flash_installation_media)
- [EFI System Partition](https://wiki.archlinux.org/index.php/EFI_System_Partition)
- [Arch Linux Partitioning](https://wiki.archlinux.org/index.php/partitioning)
- [Arch Linux LVM](https://wiki.archlinux.org/index.php/LVM)
- [Arch Linux installation with GPT, LUKS, LVM and i3](https://emanuelduss.ch/2016/03/arch-linux-installation-gpt-luks-lvm-i3/)
- [Arch Linux EFI Install Guide](http://gloriouseggroll.tv/arch-linux-efi-install-guide/)
- [How much SWAP space on a 2-4GB system?](http://serverfault.com/questions/5841/how-much-swap-space-on-a-high-memory-system)
- [I have 16GB RAM. Do I need 32GB swap?](http://askubuntu.com/a/49130)
- [Swap partition in LVM?](http://unix.stackexchange.com/questions/144586/swap-partition-in-lvm)
- [Intel graphics](https://wiki.archlinux.org/index.php/intel_graphics#Installation)
- [AMDGPU](https://wiki.archlinux.org/index.php/AMDGPU)
- [NVIDIA](https://wiki.archlinux.org/index.php/NVIDIA)
- [GNOME](https://wiki.archlinux.org/index.php/GNOME)
- [How to install Gnome on Arch Linux](http://www.muktware.io/how-to-install-gnome-on-arch-linux-arch-tutorial/)
- [libinput](https://wiki.archlinux.org/index.php/Libinput#Touchpad_tapping)
- [Touchpad Synaptics](https://wiki.archlinux.org/index.php/Touchpad_Synaptics)
- [Dell Inspiron 5547](https://wiki.archlinux.org/index.php/Dell_Inspiron_5547#Hardware)
- [How to install Yaourt on Arch Linux](http://www.ostechnix.com/install-yaourt-arch-linux/)
- [Bluetooth](https://wiki.archlinux.org/index.php/bluetooth)
- [MTP](https://wiki.archlinux.org/index.php/MTP)
- [2016 Arch Linux NetworkManager / Wifi Setup guide.](http://gloriouseggroll.tv/142-2/)
