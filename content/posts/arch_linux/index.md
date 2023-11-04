# [Arch Linux Installation](https://wiki.archlinux.org/title/installation_guide)
## Download the ISO image
Download the latest Arch Linux ISO from [here](https://archlinux.org/download/).
## Create empty disk image
Create the empty file with filled zeros of size 64GB:
```bash
dd if=/dev/zero of=bios.img bs=2G count=32
```
## [Boot the ISO image](https://wiki.archlinux.org/title/QEMU#Booting_in_UEFI_mode)
Make sure to have the `OVMF` package installed:
```bash
pacman -S ovmf
```
This package contains the UEFI firmware for QEMU.
Copy the UEFI firmware to the current directory:
```bash
cp /usr/share/edk2/x64/OVMF_VARS.fd .
```

Boot the ISO image with QEMU:
```bash
qemu-system-x86_64 \
    -enable-kvm \
    -drive file=bios.img,format=raw \
    -cdrom /usr/share/nswi106/archlinux-2023.09.01-x86_64.iso \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2-ovmf/x64/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=OVMF_VARS.fd \
    -m 2G \
    -cpu host \
    -smp 2 \
    -vnc :4232
```

## Prepare the disk
See available disks:
```bash
lsblk
```
Partition the disk:
```bash
fdisk /dev/sda
```
Than enter these commands inside the `fdisk`:
- `g` to create a new GPT partition table
- `n` to create a new partition
- `Enter` to select the default partition number
- `Enter` to select the default first sector
- `+512M` to create 512MB partition
- `t` to change the partition type
- `1` to select the first partition, if only one partition was created, it will be selected by default
- `1` to select EFI System partition type
- `n` to create a new partition
- `Enter` to select the default partition number
- `Enter` to select the default first sector
- `Enter` to select the default last sector
- `w` to write the changes to the disk
  
Check with `lsblk` that the partitions were created correctly.
Create a filesystem on the partitions:
```bash
mkfs.fat -F32 /dev/sda1
mkfs.btfs /dev/sda2
``` 
You can check the filesystems with `lsblk -f`.
Mount the filesystems:
```bash
mount /dev/sda2 /mnt
mount --mkdir /dev/sda1 /mnt/boot
```

## Install the base system
Install the essential packages:
```bash
pacstrap -K /mnt base linux linux-firmware
```

Generate the `fstab` file:
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

## Change root to the newly installed system
Chaneg root into the new system:
```bash
arch-chroot /mnt
```

## Time zone, localization, hostname and root password (optional)
Set the time zone:
```bash
ln -sf /usr/share/zoneinfo/Europe/Prague /etc/localtime
hwclock --systohc
```
Install the `vim` editor:
```bash
pacman -S vim
```
Uncomment the `en_US.UTF-8 UTF-8` and other needed locales in `/etc/locale.gen`, then generate them with:
```bash
locale-gen
```
Create the `/etc/locale.conf` file and set the `LANG` variable:
```bash
echo "LANG=en_US.UTF-8" > /etc/locale.conf
```
Set the hostname:
```bash
echo "arch" > /etc/hostname
```
Set the root password:
```bash
passwd
```

## [Bootloader](https://wiki.archlinux.org/title/GRUB)
Install the `grub` bootloader:
```bash
pacman -S grub efibootmgr
```
Install the bootloader:
```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub
```
Generate the `grub` configuration file:
```bash
grub-mkconfig -o /boot/grub/grub.cfg
```
Exit the `chroot` environment and QEMU.

## Start the new system 
Start the QEMU with the new system:
```bash
qemu-system-x86_64 \
    -enable-kvm \
    -drive file=bios.img,format=raw \
    -drive if=pflash,format=raw,readonly=on,file=/usr/share/edk2-ovmf/x64/OVMF_CODE.fd \
    -drive if=pflash,format=raw,file=OVMF_VARS.fd \
    -m 2G \
    -cpu host \
    -smp 2 \
    -vnc :4232
```