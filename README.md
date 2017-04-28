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

Note that the boot partition will be created with EFI System type since my motherboard fully supports UEFI mode. When booting in Legacy BIOS mode Linux filesystem partition type (8300) should be used.

*See References below to find out the recommended swap size.*

### Create partitions

```
# gdisk /dev/sda
```

Type `o` to create a new empty GUID partition table (GPT).

Type `n` to create the EFI partition.

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

|Logical Volume|Size           |Description     |
|--------------|---------------|----------------|
|lv_swap       |24gb           |Swap partition  |
|lv_root       |40gb           |Root partition  |
|lv_home       |Remaining space|Home partition  |

Create the swap partition.

```
# lvcreate -n lv_swap -L 24G -C y vg_linux
```

Create the root partition.

```
# lvcreate -n lv_root -L 40G vg_linux
```

Create the home partition.

```
# lvcreate -n lv_home -l 100%FREE vg_linux
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

### Swap partition

```
# swapon /dev/mapper/vg_linux-lv_swap
```

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
# ln -fs /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
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

# pacman -S bash-completion iw wpa_supplicant dialog wireless_tools rfkill wpa_actiond ifplugd mlocate openssh vim
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
# pacman -Sy yaourt
```

### Automatic switching of network profiles

Find what your interfaces are called

```
# ip link
```

Package `ifplugd` for wired interfaces: After starting and enabling `netctl-ifplugd@<interface-name>.service` DHCP profiles are started/stopped when the network cable is plugged in and out.

```
# systemctl enable netctl-ifplugd@<interface-name>.service
```

Package `wpa_actiond` for wireless interfaces: After starting and enabling `netctl-auto@<interface-name>.service` profiles are started/stopped automatically as you move from the range of one network into the range of another network (roaming).

```
# systemctl enable netctl-auto@<interface-name>.service
```

### Install and configure sudo

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
# pacman -S networkmanager network-manager-applet dhclient
```

You must ensure that no other service that wants to configure the network is running; in fact, multiple networking services will conflict. So first let’s find our devices:

```
# ip link
```

Anything starting with *enp* is an ethernet device.

Anything starting with *wlp* is a wireless device.

You can find a list of the currently running services with `systemctl --type=service` and then stop them.

```
# systemctl --type=service
```

In my case, two network services have to be disabled.

```
# systemctl disable netctl-ifplugd@enp1s0.service
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

## Enabling tap-to-click

Tap-to-click is disabled in GDM (and GNOME) by default, but you can easily enable it with a dconf setting.

If you want to do this under X, you have to first set up correct X server access permissions.

```
# xhost +SI:localuser:gdm
```

To directly enable tap-to-click, use:

```
# sudo -u gdm gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
```

If you prefer to do this with a GUI, use:

# sudo -u gdm dconf-editor
```

To check the if it was set correctly, use:

```
# sudo -u gdm gsettings get org.gnome.desktop.peripherals.touchpad tap-to-click
```

If you get the error `dconf-WARNING **: failed to commit changes to dconf: Error spawning command line`, make sure dbus is running:

```
# sudo -u gdm dbus-launch gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
```

## Enabling Hibernation

In order to use hibernation, you need to create a swap partition or file. You will need to point the kernel to your swap using the `resume=` kernel parameter, which is configured via the boot loader. You will also need to configure the initramfs. This tells the kernel to attempt resuming from the specified swap in early userspace.

The kernel parameter resume=swap_partition has to be used. Either the name the kernel assigns to the partition or its UUID can be used as swap_partition. For example:

* `resume=/dev/sda1`

* `resume=UUID=4209c845-f495-4c43-8a03-5363dd433153`

* `resume=/dev/mapper/archVolumeGroup-archLogicVolume` -- example if using LVM

The configuration depends on the used boot loader. The Arch Linux installation medium uses Syslinux for BIOS systems, and systemd-boot for UEFI systems.

**Syslinux**

Edit `/boot/syslinux/syslinux.cfg` and add them to the `APPEND` line:

```
APPEND ... resume=vg_linux-lv_swap
```

**systemd-boot**

Edit `/boot/loader/entries/arch.conf` (assuming you set up your EFI System Partition) and add them to the options line:

```
options ... resume=vg_linux-lv_swap
```

**Configure the initramfs**

* When an initramfs with the base hook is used, which is the default, the resume hook is required in `/etc/mkinitcpio.conf`. Whether by label or by UUID, the swap partition is referred to with a udev device node, so the resume hook must go after the udev hook. This example was made starting from the default hook configuration:

```
HOOKS="base udev resume autodetect modconf block filesystems keyboard fsck"
```

Note: LVM users should add the resume hook after lvm2.

Remember to rebuild the initramfs for these changes to take effect.

```
# mkinitcpio -p linux
```

* When an initramfs with the systemd hook is used, a resume mechanism is already provided, and no further hooks need to be added.

## SSD Optimization

Users need to be certain that their SSD supports TRIM before attempting to use it. To verify TRIM support, run:

```
# lsblk -D
```

And check the values of DISC-GRAN and DISC-MAX columns. Non-zero values indicate TRIM support.

**Periodic TRIM**

While it is possible to enable continuous TRIM in Linux, this can actually negatively affect performance because of the additional overhead on normal file operations. A gentler alternative is to configure periodic TRIM. This configures the operating system to TRIM the drive on a schedule instead of as a necessary component of regular file operations. In almost all cases it provides the same benefits of continuous TRIM without the performance hit.

Install the `util-linux` package.

```
# pacman -S util-linux
```

To schedule a weekly TRIM of all attached capable drives, enable the timer.

```
sudo systemctl enable fstrim.timer
```

**Continuous TRIM**

Performing TRIM on every deletion can be costly and can have a negative impact on the performance of the drive.

Using the discard option for a mount in `/etc/fstab` enables continuous TRIM in device operations.

```
/dev/sda2  /boot       ext4  defaults,noatime,discard   0  2
/dev/sda1  /boot/efi   vfat  defaults,noatime,discard   0  2
/dev/sda3  /           ext4  defaults,noatime,discard   0  2
```

Continuous TRIM can be disabled by removing the discard option for a mount in `/etc/fstab`.

```
/dev/sda2  /boot       ext4  defaults,noatime   0  2
/dev/sda1  /boot/efi   vfat  defaults,noatime   0  2
/dev/sda3  /           ext4  defaults,noatime   0  2
```

**Enabling TRIM support on LVM**

Edit the LVM configuration file `/etc/lvm/lvm.conf` and enable the option `issue_discards`.

```
# [...]
devices {
   # [...]
   issue_discards = 1
   # [...]
}
# [...]
```

## Install optional software

### VirtualBox

Install the core packages.
```
# pacman -S virtualbox virtualbox-host-modules-arch virtualbox-guest-iso
```

Load the VirtualBox kernel modules.
```
# modprobe vboxdrv boxnetadp vboxnetflt vboxpci
```

Add users that will be authorized to access host USB devices in guest to the `vboxusers` group.
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

### Enable OpenSSH daemon

```
# systemctl enable sshd.socket
```

## Tools

### Docker

Install the `docker` package.

```
# pacman -S docker
```

Next start and enable `docker.service`.

```
# systemctl start docker.service
# systemctl enable docker.service
```

If you want to be able to run docker as a regular user, add yourself to the docker group.

```
# gpasswd -a myuser docker
```

Then re-login or to make your current user session aware of this new group.
```
# newgrp docker
```

## Troubleshooting

### Xorg backend

The Wayland backend is used by default and the Xorg backend is used only if the Wayland backend cannot be started. As the Wayland backend has been [reported](https://bugzilla.redhat.com/show_bug.cgi?id=1199890) to cause problems for some users, use of the Xorg backend may be necessary.
To use the Xorg backend by default, edit the `/etc/gdm/custom.conf` file and uncomment the following line:

```
#WaylandEnable=false
```

### Possibly missing firmware for module

How to deal with initramfs’s rebuild warnings (after a kernel update) regarding “Possibly missing firmware for module“…
When initramfs are being rebuild after a kernel update, some kernel warnings may appear, e.g.:
```
==> WARNING: Possibly missing firmware for module: wd719x
==> WARNING: Possibly missing firmware for module: aic94xx
```

Search for firmware driver modules in AUR, e.g.:
```
yaourt -Ss wd719x
yaourt -Ss aic94xx
```

Install those that were found, e.g.:
```
yaourt -S wd719x-firmware aic94xx-firmware
```

Recompile kernel:
```
mkinitcpio -p linux
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
- [GDM](https://wiki.archlinux.org/index.php/GDM)
- [libinput](https://wiki.archlinux.org/index.php/Libinput#Touchpad_tapping)
- [Touchpad Synaptics](https://wiki.archlinux.org/index.php/Touchpad_Synaptics)
- [Dell Inspiron 5547](https://wiki.archlinux.org/index.php/Dell_Inspiron_5547#Hardware)
- [How to install Yaourt on Arch Linux](http://www.ostechnix.com/install-yaourt-arch-linux/)
- [Bluetooth](https://wiki.archlinux.org/index.php/bluetooth)
- [MTP](https://wiki.archlinux.org/index.php/MTP)
- [2016 Arch Linux NetworkManager / Wifi Setup guide.](http://gloriouseggroll.tv/142-2/)
- [How to properly activate TRIM for your SSD on Linux: fstrim, lvm and dm-crypt](http://blog.neutrino.es/2013/howto-properly-activate-trim-for-your-ssd-on-linux-fstrim-lvm-and-dmcrypt/)
- [SSDOptimization](https://wiki.debian.org/SSDOptimization)
- [How To Configure Periodic TRIM for SSD Storage on Linux Servers](https://www.digitalocean.com/community/tutorials/how-to-configure-periodic-trim-for-ssd-storage-on-linux-servers)
- [Arch Linux - SSD Trim on encrypted LVM volumes](http://ggarcia.me/2016/10/11/arch-linux-ssd-trim.html)