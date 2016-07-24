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
#lvdisplay
```

Our logical volumes should now be located in `/dev/mapper/` and `/dev/vg_linux`.
If you cannot find them, use the next commands to bring up the module for creating device nodes and to make volume groups available:

```
# modprobe dm_mod
# vgscan
# vgchange -ay
```

### Create file systems

- EFI System Partition (ESP)
```
# mkfs.fat -F32 /dev/sda1
```

- Swap partition
```
# mkswap /dev/mapper/vg_linux-lv_swap
```

- Root partition
```
# mkfs.ext4 /dev/mapper/vg_linux-lv_root
```

- Home partition
```
# mkfs.ext4 /dev/mapper/vg_linux-lv_home
```

### Mount file systems

- Root partition
```
# mount /dev/mapper/vg_linux-lv_root /mnt
```

- Home partition
```
# mkdir /mnt/home
# mount /dev/mapper/vg_linux-lv_home /mnt/home
```

- EFI System Partition (ESP)
```
# mkdir /mnt/boot
# mount /dev/sda1 /mnt/boot
```

Confirm that the file systems were mounted correctly.
```
lsblk
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
