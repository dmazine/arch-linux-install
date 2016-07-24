# Arch Linux Install

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

#<ip-address>	<hostname.domain.org>	<hostname>
127.0.0.1	localhost.localdomain	localhost	 myhostname
::1		localhost.localdomain	localhost	 myhostname
```


## References

- [Arch Linux Installation guide](https://wiki.archlinux.org/index.php/installation_guide#Pre-installation)
- [EFI System Partition](https://wiki.archlinux.org/index.php/EFI_System_Partition)
- [Arch Linux Partitioning](https://wiki.archlinux.org/index.php/partitioning)
- [Arch Linux LVM](https://wiki.archlinux.org/index.php/LVM)
- [Arch Linux EFI Install Guide](http://gloriouseggroll.tv/arch-linux-efi-install-guide/)
- [How much SWAP space on a 2-4GB system?](http://serverfault.com/questions/5841/how-much-swap-space-on-a-high-memory-system)
- [I have 16GB RAM. Do I need 32GB swap?](http://askubuntu.com/a/49130)
- [Swap partition in LVM?](http://unix.stackexchange.com/questions/144586/swap-partition-in-lvm)
