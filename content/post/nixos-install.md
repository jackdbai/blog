---
publishdate: 2022-03-11T00:32:34-05:00
title: "NixOS Install"
author: "Jack"
description: "My reference guide for installing NixOS"
featured: false
draft: false
---

My guide for installing NixOS, will be updated as necessary.  

### Prepare Disks  
```bash
# Set new disklabel
parted -s /dev/sdX mklabel msdos

# Make EFI partition
parted -s -a optimal /dev/sdX mkpart "primary" "fat16" "0%" "1024MiB"
parted -s /dev/sdX set 1 boot on
parted -s /dev/sdX print
parted -s /dev/sdX align-check optimal 1

# Make LVM partition
parted -s -a optimal /dev/sdX mkpart "primary" "ext4" "1024MiB" "100%"
parted -s /dev/sdX set 2 lvm on
```

### Set Up LVM  
```bash
# Format and open LVM partition
cryptsetup luksFormat /dev/sdX2
cryptsetup luksOpen /dev/sdX2 lvm-system

# Set up physical, virtual, and logical LVM volumes
pvcreate /dev/mapper/lvm-system
vgcreate lvmSystem /dev/mapper/lvm-system
lvcreate -L 16G lvmSystem -n volSwap
lvcreate -l +100%FREE lvmSystem -n volRoot
```

### Format and Mount  
```bash
# Format
mkfs.fat -n BOOT /dev/sdX1
mkswap /dev/lvmSystem/volSwap
mkfs.ext4 -L volRoot /dev/lvmSystem/volRoot

# Mount
swapon /dev/lvmSystem/volSwap
mount /dev/lvmSystem/volRoot /mnt
mkdir /mnt/boot
mount /dev/sdX1 /mnt/boot
```

### NixOS Configuration
Once everything is mounted, generate NixOS configurations:
```bash
nixos-generate-config --root /mnt
```

Back up the existing `configuration.nix` and copy in the new one:
```bash
mv /mnt/etc/nixos/configuration.nix /mnt/etc/nixos/configuration.nix.old
wget https://blog.jackdbai.dev/configuration.nix -P /mnt/etc/
```

Be sure to grab the UUID of your LVM partition by using `lsblk --fs`:
```
NAME FSTYPE FSVER LABEL UUID                                   FSAVAIL FSUSE% MOUNTPOINTS
sdX                                                                           
├─sdX1
│    vfat   FAT16 BOOT                                                        /boot
└─sdX2
     crypto 2           {SDX2 UUID, THE UUID WE WANT}                  
  └─luksroot
     LVM2_m LVM2        {lvm uuid}                
    ├─lvmSystem-volSwap
    │  swap   1           {swap uuid}                  
    └─lvmSystem-volRoot
       ext4   1.0   volRoot
                          {root uuid}                  /nix/store
```  
Be sure to swap that UUID in otherwise Grub won't know what to unlock. Take some time to tailor the rest of the config to your liking.  
To install just:
```bash
nixos-install
```

### References
- [Guide for Encrypted NixOS Install](https://gist.github.com/martijnvermaat/76f2e24d0239470dd71050358b4d5134)
- [Artix Encrypted Guide for Partition Scheme](https://wiki.artixlinux.org/Main/InstallationWithFullDiskEncryption)
- [Documentation for the pesky luksroot issue](https://github.com/NixOS/nixpkgs/issues/113337)
