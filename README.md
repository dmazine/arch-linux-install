# Arch Linux Installation Guide on a UEFI system

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

## Pre-Installation

### Set the keyboard layout

```
# loadkeys br-abnt2
```

Available choices can be listed with ls `/usr/share/kbd/keymaps/**/*.map.gz`.

### Verify the boot mode

If UEFI mode is enabled on an UEFI motherboard, Archiso will boot Arch Linux accordingly via systemd-boot. To verify this, list the efivars directory:

```
# ls /sys/firmware/efi/efivars
```

If the directory does not exist, the system may be booted in BIOS or CSM mode. Refer to your motherboard's manual for details.

### Connect to the Internet

The installation image enables the dhcpcd daemon on boot for wired devices, and will attempt to start a connection. Verify internet connectivity is available, for example with ping:

```
ping -c 3 www.google.com
```

`wifi-menu` can be used to connect to a wireless network:

```
wifi-menu
```

### Update the system clock

```
# timedatectl set-ntp true
```

### Partition the disks

Identify the disk names with `lsblk` (results ending in rom, loop or airoot can be ignored).

In my case, I'm going to install Arch on `/dev/sda` and create two partitions:

|Partition|Type             |Size                |Description     |
|---------|-----------------|--------------------|----------------|
|sda1     |EFI System (ef00)|512mb               |Boot partition  |
|sda2     |Linux LVM (8e00) |Remaining free space|LVM partition   |

Note that the boot partition will be created with EFI System type since Arch will be installed on a motherboard that fully supports UEFI mode.

*See References below to find out the recommended swap size.*

#### Create partitions

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

#### Create LVM physical volume

```
# pvcreate /dev/sda2
```

Confirm that the physical volume was created correctly.

```
# pvdisplay
```

#### Create LVM volume group

```
# vgcreate vg_linux /dev/sda2
```

Confirm that the volume group was created correctly.

```
# vgdisplay
```

#### Create LVM logical volumes

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

### Format the partitions

#### EFI System Partition (ESP)

```
# mkfs.fat -F32 /dev/sda1
```

#### Swap partition

```
# mkswap /dev/mapper/vg_linux-lv_swap
```

#### Root partition

```
# mkfs.ext4 /dev/mapper/vg_linux-lv_root
```

#### Home partition

```
# mkfs.ext4 /dev/mapper/vg_linux-lv_home
```

### Mount the partitions

#### Swap partition

```
# swapon /dev/mapper/vg_linux-lv_swap
```

#### Root partition

```
# mount /dev/mapper/vg_linux-lv_root /mnt
```

#### Home partition

```
# mkdir /mnt/home
# mount /dev/mapper/vg_linux-lv_home /mnt/home
```

#### Boot Partition

```
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

Confirm that the file systems were mounted correctly.

```
# lsblk
```

## Installation

### Rank the mirrors by speed

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

### Install the base system

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
# ln -sf /usr/share/zoneinfo/America/Sao_Paulo /etc/localtime
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
127.0.0.1        localhost.localdomain    localhost
::1              localhost.localdomain    localhost

127.0.0.1        myhostname.mydomain      myhostname
::1              myhostname.mydomain      myhostname
```

### Network configuration

The basic installation procedure typically has a functional network configuration. 

For Wireless configuration, install the `iw` and `wpa_supplicant` packages, as well as needed firmware packages. Optionally install `dialog` for usage of `wifi-menu`.

```
# pacman -S iw wpa_supplicant dialog
```

### Initramfs

Creating a new initramfs is usually not required, because `mkinitcpio` was run on installation of the linux package with `pacstrap`. However, since the root filesystem is on LVM, it will be needed to enable the appropriate `mkinitcpio` hooks, otherwise your system might not boot.

Edit the file `/etc/mkinitcpio.conf` and insert `lvm2` between block and filesystems like so:

```
HOOKS="base udev ... block lvm2 filesystems"
```

Recreate the initramfs image:

```
# mkinitcpio -p linux
```

### Set root password

```
# passwd
```

### Boot loader

Install the boot loader.

```
# bootctl install
```

Arch will be installed on a motherboard that fully supports UEFI mode, thus Arch Linux installation medium uses `systemd-boot` boot manager.

Create a boot entry in `/boot/loader/entries/arch.conf`

```
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/mapper/vg_linux-lv_root rw
```

Update the default entry in `/boot/loader/loader.conf`

```
timeout 2
default arch
```

If you have an Intel CPU, install the `intel-ucode` package in addition, and enable microcode updates.

```
# pacman -S intel-ucode
```

Use the initrd option twice in `/boot/loader/entries/arch.conf`.

```
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options ...
```

### Enable Hibernation (Optional)

In order to use hibernation, you will need to point the kernel to your swap using the `resume=` kernel parameter, which is configured via the boot loader. The configuration depends on the used boot loader.

Arch will be installed on a motherboard that fully supports UEFI mode, thus Arch Linux installation medium uses `systemd-boot` boot manager.

Edit the `/boot/loader/entries/arch.conf` file and add the `resume=` kernel parameter to the options line:

```
options root=/dev/mapper/vg_linux-lv_root rw resume=/dev/mapper/vg_linux-lv_swap
```

It will be also needed to configure the `initramfs`. This tells the kernel to attempt resuming from the specified swap in early userspace.

Edit the `/etc/mkinitcpio.conf` file and add the `resume` hook. Whether by label or by UUID, the swap partition is referred to with a udev device node, so the resume hook must go after the udev hook. LVM users should add the `resume` hook after `lvm2`.

```
HOOKS="base udev autodetect modconf block lvm2 resume filesystems keyboard fsck"
```

Recreate the initramfs image:

```
# mkinitcpio -p linux
```

### SSD Optimization (Optional)

Users need to be certain that their SSD supports TRIM before attempting to use it. To verify TRIM support, run:

```
# lsblk -D
```

And check the values of `DISC-GRAN` and `DISC-MAX` columns. Non-zero values indicate TRIM support.

**Periodic TRIM**

While it is possible to enable continuous TRIM in Linux, this can actually negatively affect performance because of the additional overhead on normal file operations. A gentler alternative is to configure periodic TRIM. This configures the operating system to TRIM the drive on a schedule instead of as a necessary component of regular file operations. In almost all cases it provides the same benefits of continuous TRIM without the performance hit.

Install the `util-linux` package.

```
# pacman -S util-linux
```

To schedule a weekly TRIM of all attached capable drives, enable the timer.

```
# systemctl enable fstrim.timer
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

## Reboot

Exit the chroot environment:

```
# exit
```

Unmount all the partitions:

```
# umount -R /mnt
```

Restart the machine:

```
# reboot
```

## Post-installation

### Install and configure sudo

```
# pacman -S sudo
```

Run the `visudo` to edit the `/etc/sudoers` file

```
# visudo
```

Grant sudo access to users in the group wheel when enabled.

```
## Allows people in group wheel to run all commands
%wheel    ALL=(ALL)    ALL
```

### Create user

```
# useradd -m -g users -G wheel -s /bin/bash myuser
# passwd myuser
```

### Enable multilib repository in `/etc/pacman.conf`

```
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Pacman has a color option. Uncomment the `Color` line in `/etc/pacman.conf` to enable color output in console.

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

### Display driver

#### Intel

Install the `mesa` package, which provides the DRI driver for 3D acceleration.

* For 32-bit application support on x86_64, also install `lib32-mesa` from multilib.
* For the DDX driver (which provides 2D acceleration in Xorg), install the `xf86-video-intel package`. **(Often not recommended, see note below.)**
* For Vulkan support (Ivy Bridge and newer), install the `vulkan-intel package`.

**Note:** Some (Debian & Ubuntu, Fedora, KDE) recommend not installing the xf86-video-intel driver, and instead falling back on the modesetting driver for fourth generation and newer GPUs. See [It is probably time to ditch xf86-video-intel](https://www.reddit.com/r/archlinux/comments/4cojj9/it_is_probably_time_to_ditch_xf86videointel/) and [Intel vs. Modesetting X.Org DDX Performance Impact](http://www.phoronix.com/scan.php?page=article&item=intel-modesetting-2017&num=1).

```
# pacman -S xf86-video-intel mesa lib32-mesa
```

#### ATI/AMD

In order for video acceleration to work, and often to expose all the modes that the GPU can set, a proper video driver is required:

|Brand  |Type       |Driver            |OpenGL            |OpenGL (Multilib)       |Documentation                                                        |
|-------|-----------|------------------|------------------|------------------------|---------------------------------------------------------------------|
|AMD/ATI|Open source|xf86-video-amdgpu |mesa              |lib32-mesa              |[AMDGPU](https://wiki.archlinux.org/index.php/AMDGPU)                |
|AMD/ATI|Open source|xf86-video-ati    |mesa              |lib32-mesa              |[ATI](https://wiki.archlinux.org/index.php/ATI)                      |
|AMD/ATI|Proprietary|catalyst          |catalyst-libgl    |lib32-catalyst-libgl    |[AMD Catalyst](https://wiki.archlinux.org/index.php/AMD_Catalyst)    |
|Intel  |Open source|xf86-video-intel  |mesa              |lib32-mesa              |[Intel graphics](https://wiki.archlinux.org/index.php/Intel_graphics)|
|Nvidia |Open source|xf86-video-nouveau|mesa              |lib32-mesa              |[Nouveau](https://wiki.archlinux.org/index.php/Nouveau)              |
|Nvidia |Proprietary|nvidia            |nvidia-utils      |lib32-nvidia-utils      |[NVIDIA](https://wiki.archlinux.org/index.php/NVIDIA)                |
|Nvidia |Proprietary|nvidia-340xx      |nvidia-340xx-utils|lib32-nvidia-340xx-utils|[NVIDIA](https://wiki.archlinux.org/index.php/NVIDIA)                |
|Nvidia |Proprietary|nvidia-304xx      |nvidia-304xx-utils|lib32-nvidia-304xx-utils|[NVIDIA](https://wiki.archlinux.org/index.php/NVIDIA)                |

**Note:** For NVIDIA Optimus enabled laptop which uses an integrated video card combined with a dedicated GPU, see [NVIDIA Optimus](https://wiki.archlinux.org/index.php/NVIDIA_Optimus) or [Bumblebee](https://wiki.archlinux.org/index.php/Bumblebee).

**Tip:** If you are trying to figure out what drivers to install for your card, [this spreadsheet](https://goo.gl/R0Te9T) created by fellow Linux users may help you.

**AMD**

|GPU architecture|Radeon cards     |Open-source driver|Proprietary driver |
|----------------|-----------------|------------------|-------------------|
|GCN 4 and newer |various          |AMDGPU            |AMDGPU PRO         |
|GCN 3           |various          |AMDGPU            |Catalyst/AMDGPU PRO|
|GCN 2*          |various          |AMDGPU/ATI        |Catalyst           |
|GCN 1*          |various          |AMDGPU/ATI        |Catalyst           |
|TeraScale 2&3   |HD 5000 - HD 6000|ATI               |Catalyst           |
|TeraScale 1     |HD 2000 - HD 4000|ATI               |Catalyst legacy    |
|Older           |X1000 and older  |ATI               |not available      |

*: Experimental AMDGPU support


```
# pacman -S xf86-video-amdgpu lib32-mesa mesa-vdpau lib32-mesa-vdpau
```

#### Nvidia

Install the appropriate driver for your card:

* For GeForce 400 series cards and newer [NVCx and newer], install the `nvidia` or `nvidia-lts package`. If these packages do not work, `nvidia-beta` may have a newer driver version that offers support.
* For GeForce 8000/9000, ION and 100-300 series cards [NV5x, NV8x, NV9x and NVAx] from around 2006-2010, install the `nvidia-340xx` or `nvidia-340xx-lts` package.
* For GeForce 6000/7000 series cards [NV4x and NV6x] from around 2004-2006, install the `nvidia-304xx` or `nvidia-304xx-lts` package.
* For even older cards, have a look at [#Unsupported drivers](https://wiki.archlinux.org/index.php/NVIDIA#Unsupported_drivers).

For 32-bit application support on x86_64, you must also install the equivalent lib32 package from the multilib repository (e.g. `lib32-nvidia-utils`, `lib32-nvidia-340xx-utils` or `lib32-nvidia-304xx-utils`).

Reboot. The nvidia package contains a file which blacklists the nouveau module, so rebooting is necessary.

The NVIDIA package includes an automatic configuration tool to create an Xorg server configuration file (xorg.conf) and can be run by:

```
# nvidia-xconfig
```

This command will auto-detect and create (or edit, if already present) the `/etc/X11/xorg.conf` configuration according to present hardware.

If there are instances of DRI, ensure they are commented out:

```
#    Load        "dri"
```

The `nvidia-settings` tool lets you configure many options using either CLI or GUI.

#### Hybrid graphics

PRIME is a technology used to manage hybrid graphics found on recent laptops (Optimus for NVIDIA, AMD Dynamic Switchable Graphics for Radeon). PRIME GPU offloading and Reverse PRIME is an attempt to support muxless hybrid graphics in the Linux kernel.

Remove any closed-source graphic drivers and replace them with the open source equivalent:

* xf86-video-nouveau

* xf86-video-ati

* xf86-video-amdgpu

* xf86-video-intel

Reboot and check the list of attached graphic drivers:

```
# xrandr --listproviders
```

We can see that there are two graphic cards: Intel, the integrated card (id 0x7d), and Radeon, the discrete card (id 0x56), which should be used for GPU-intensive applications.

### Audio driver

```
# pacman -S pulseaudio pulseaudio-alsa lib32-libpulse lib32-alsa-plugins
```

### Bluetooth

```
# pacman -S bluez bluez-utils
# systemctl enable bluetooth
```

### Automounting removable media or network shares

```
# pacman -S autofs
```

### NTFS file system including read and write support

```
# pacman -S ntfs-3g
```

### Desktop Environment

#### GNOME desktop

```
# pacman -S gnome
```

Install further GNOME applications, including an archive manager, disk manager, text editor, and a set of games.

```
# pacman -S gnome-extra
```

Enable `gdm.service` to start GDM at boot time

```
# systemctl enable gdm.service
```

Tap-to-click is disabled in GDM (and GNOME) by default, but you can easily enable it with a dconf setting.

To enable tap-to-click, use:

```
# sudo -u gdm gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
```

If you get the error `dconf-WARNING **: failed to commit changes to dconf: Error spawning command line`, make sure dbus is running:

```
# sudo -u gdm dbus-launch gsettings set org.gnome.desktop.peripherals.touchpad tap-to-click true
```

To check the if it was set correctly, use:

```
# sudo -u gdm gsettings get org.gnome.desktop.peripherals.touchpad tap-to-click
```

### Network Manager

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

In my case, one network service have to be disabled.

```
# systemctl disable netctl-ifplugd@enp1s0.service
# systemctl disable netctl-auto@wlp2s0.service
```

Enable network manager:

```
# systemctl enable NetworkManager.service
```











### Install additional packages

### Archive formats

```
# pacman -S p7zip unrar tar rsync
```

### Chromium Web Browser

```
# pacman -S chromium
```

### DBeaver

```
# yaourt -S dbeaver
```

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

### Eclipse

```
# pacman -S eclipse-jee
```

### File index and search

```
# pacman -S mlocate
# updatedb
```

### Firefox Web Browser

```
# pacman -S firefox
```

### Java

```
# pacman -S jdk8-openjdk openjdk8-doc
```

### Evolution Spell Check

```
# pacman -S aspell-en aspell-pt
```

### HP Printers

```
# pacman -S hplip
```

### Libre Office

```
# pacman -S libreoffice
```

### NodeJS

```
# pacman -S nodejs
```

### OpenSSH

```
# pacman -S openssh
# systemctl enable sshd.socket
```

### Printing Service

```
# pacman -S cups cups-pdf system-config-printer
# systemctl enable org.cups.cupsd.service
```

### Skype

```
# yaourt -S skypeforlinux
```

### SoapUI

```
# yaourt -S soapui
```

### Spotify

```
# yaourt -S spotify
```

If you encounter the following error while installing, the public key needs to be downloaded.

```
==> Verifying source file signatures with gpg...
    curl-7.54.0.tar.gz ... FAILED (unknown public key 5CC908FDB71E12C2)
==> ERROR: One or more PGP signatures could not be verified!
==> ERROR: Makepkg was unable to build libcurl-openssl-1.0.
```

To download the public key execute the following command:

```
# gpg --recv-keys 5CC908FDB71E12C2
```

### Sublime Text 3

```
# yaourt -S sublime-text-dev
```

### VI Improved

```
# pacman -S vim
```

### VirtualBox

Install the core packages.

```
# pacman -S virtualbox virtualbox-host-modules-arch virtualbox-guest-iso
```

Load the VirtualBox kernel modules.

```
# modprobe vboxdrv vboxnetadp vboxnetflt vboxpci
```

Add users that will be authorized to access host USB devices in guest to the `vboxusers` group.

```
# usermod -a -G vboxusers <login>
```

### VLC Media Player

```
# pacman -S vlc
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
- [Arch Linux Post Installation (30 Things to do after Installing Arch Linux)](http://www.2daygeek.com/arch-linux-post-installation-30-things-to-do-after-installing-arch-linux/#)
