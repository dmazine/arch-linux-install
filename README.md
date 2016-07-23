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
Identify the disk names with `lsblk`

### Wiping the existing partition table
```
gdisk /dev/sdX
```
Type `x` to access extra functionalities

Type `z` to destroy (zap) GPT data structures



```
```
