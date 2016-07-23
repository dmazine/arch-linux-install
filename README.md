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
Test it using `ping -c 3 www.google.com`

## Update the system clock
```
# timedatectl set-ntp true
```

## Partition the disks
Identify the disk names with `lsblk` (results ending in rom, loop or airoot can be ignored)

In my case, I'm going to install Arch on `/dev/sda`

Wipe the existing partition table
```
gdisk /dev/sda

Type x to access extra functionalities

Type z to destroy (zap) GPT data structures
```

We are going to create three partitions:

|Partition|Type             |Size                |Description     |
|---------|-----------------|--------------------|----------------|
|sda1     |EFI System (ef00)|512mb               |Boot partition  |
|sda2     |Linux Swap (8200)|24Ggb               |Swap partition  |
|sda3     |Linux LVM (8e00) |Remaining free space|LVM partition   |

*See References below to find out the recommended swap size.*

Create new partitions
```
gdisk /dev/sda

Type n to create the boot partition
Partition number: 1
First sector: Leave this blank
Size in sectors: +512M
Hex code or GUID: ef00

Type n to create the swap partition
Partition number: 2
First sector: Leave this blank
Size in sectors: +24G
Hex code or GUID: 8200

Type n to create the LVM partition
Partition number: 3
First sector: Leave this blank
Size in sectors: Leave this blank
Hex code or GUID: 8e00

Type p to print the partition table and check if it is correct

Type w to write the partition table to disk and exit
```


## References

[Arch Linux Installation guide](https://wiki.archlinux.org/index.php/installation_guide#Pre-installation)
[Arch Linux EFI Install Guide](http://gloriouseggroll.tv/arch-linux-efi-install-guide/)
[Arch Linux LVM](https://wiki.archlinux.org/index.php/LVM)
[How much SWAP space on a 2-4GB system?](http://serverfault.com/questions/5841/how-much-swap-space-on-a-high-memory-system)
[I have 16GB RAM. Do I need 32GB swap?](http://askubuntu.com/a/49130)
