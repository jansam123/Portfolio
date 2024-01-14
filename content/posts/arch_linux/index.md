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
mkfs.btrfs /dev/sda2
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

### Time synchronization
Use the `timedatectl` command to ensure the system clock is accurate:
```bash
timedatectl set-ntp true
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

# Network Setup
In the further we assume that we have a machine `a` which has htwo VMs running `ns1` and `gw` and we want to connect them to the internet, such that `ns1` can access the internet through `gw`.

## VDE Switch
Create a `vde_switch` network:
```bash
/usr/bin/vde_switch -sock /home/jankovys/vde/sw1/comm -daemon
```
Or create a service that will start the `vde_switch` automatically. Create a file `/etc/systemd/system/sw1.service` or `~/.config/systemd/user/sw1.service` with the following content:
```conf
[Unit]
Description=Vde Switch
After=network.target

[Service]
ExecStart=/usr/bin/vde_switch -sock /home/jankovys/vde/sw1/comm -daemon
Type=forking
Restart=always
RestartSec=1

[Install]
WantedBy=default.target
```
Then enable and start the service:
```bash
systemctl --user enable --now sw1.service
```
## VMs setup
### `ns1`
Connect the VM to the `vde_switch` network with option
```bash
-nic vde,mac=de:ad:be:ef:20:03,sock="$HOME/vde/sw1/comm"
```
Create a network interface on the `ns1` by creating a file `/etc/systemd/network/ens3.network` with the following content:
```conf
[Match]
Name=ens3

[Network]
Address=10.0.42.2/24
Gateway=10.0.42.1
DNS=8.8.8.8
```
Then enable and start the service:
```bash
systemctl enable --now systemd-networkd
```

You can check the network configuration with:
```bash
ip addr show
ip route show
```
Add a nameserver to `/etc/resolv.conf`:
```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

### `gw`
Connect the VM to the `vde_switch` network and external network with options
```bash
-nic vde,mac=de:ad:be:ef:20:01,sock=/var/run/vde/backbone/comm \
-nic vde,mac=de:ad:be:ef:20:02,sock="$HOME/vde/sw1/comm" \
```
Create a `ens3` network interface connecetd to the external internet on the `gw` by creating a file `/etc/systemd/network/ens3.network` with the following content:
```conf
[Match]
Name=ens3

[Network]
Address=10.0.0.42/24
Gateway=10.0.0.1
DNS=10.0.0.1
```
Additionaly create a `ens4` network interface connected to the `vde_switch` network by creating a file `/etc/systemd/network/ens4.network` with the following content:
```conf
[Match]
Name=ens4

[Network]
Address=10.0.42.1/24
``` 
Then enable and start the service:
```bash
systemctl enable --now systemd-networkd
```

Add a nameserver to `/etc/resolv.conf`:
```bash
echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

### Setp up masquerading
Add the `IPMasquerade=yes` line to `/etc/systemd/network/ens4.network` and `/etc/systemd/network/ens5.network` on the `gw` to enable fowarding and masquerading of the traffic from `ens4` and `ens5` to the Internet:
```conf
[Match]
Name=ens4

[Network]
Address=10.0.42.1/24
IPMasquerade=yes
```
```conf
[Match]
Name=ens5

[Network]
Address=10.0.142.1/24
IPMasquerade=yes
```
Then restart the `systemd-networkd` service:
```bash
systemctl restart systemd-networkd
``` 


# VM base setup
Create users:
```bash
useradd -m jankovys
useradd -m eval
```
Change the password for the users:
```bash
passwd jankovys
passwd eval
```
Install the `sudo` package:
```bash
pacman -S sudo
```
Add the users to be able to use `sudo` without password by editing the `/etc/sudoers` file
```bash
pacman -S vi
visudo
```
and adding the following lines:
```conf
## User privilege specification
jankovys ALL=(ALL) ALL
eval ALL=(ALL) ALL
##
Defaults:jankovys !authenticate
Defaults:evel !authenticate
```

Install the `openssh` package:
```bash
pacman -S openssh
```
Enable the `sshd` service:
```bash
systemctl enable --now sshd
```
Add the puclib key to the `.ssh/authorized_keys` file:
```bash
ssh-ed25519 .....
```
Disable passwrd authentication by editing the `/etc/ssh/sshd_config` file and setting the following options:
```conf
PasswordAuthentication no
# ChallengeResponseAuthentication no
```
Restart the `sshd` service:
```bash
systemctl restart sshd
```

# Snapper setup
## Setp up `cronie`
Install `cronie` package:
```bash
pacman -S cronie
```
Enable and start the `cronie` service:
```bash
systemctl enable --now cronie
```
The `cron` jobs can be edited by 
```bash
crontab -e
```
and the list of the jobs can be displayed by
```bash
crontab -l 
```

## Setup `snapper`
Snapper is a tool for creating snapshots of the filesystem. 
Install the `snapper` package:
```bash
pacman -S snapper
```

Generate the default configuration:
```bash
snapper -c root create-config /
```
This will create a configuration file `/etc/snapper/configs/root`, which you can edit to your needs.
<details>
  <summary><i>Expand to see the config file</i></summary>

```conf
# subvolume to snapshot
SUBVOLUME="/"

# filesystem type
FSTYPE="btrfs"


# btrfs qgroup for space aware cleanup algorithms
QGROUP=""


# fraction or absolute size of the filesystems space the snapshots may use
SPACE_LIMIT="0.5"

# fraction or absolute size of the filesystems space that should be free
FREE_LIMIT="0.2"


# users and groups allowed to work with config
ALLOW_USERS=""
ALLOW_GROUPS=""

# sync users and groups from ALLOW_USERS and ALLOW_GROUPS to .snapshots
# directory
SYNC_ACL="no"


# start comparing pre- and post-snapshot in background after creating
# post-snapshot
BACKGROUND_COMPARISON="yes"


# run daily number cleanup
NUMBER_CLEANUP="yes"

# limit for number cleanup
NUMBER_MIN_AGE="1800"
NUMBER_LIMIT="30"
NUMBER_LIMIT_IMPORTANT="30"

# create hourly snapshots
TIMELINE_CREATE="yes"

# cleanup hourly snapshots after some time
TIMELINE_CLEANUP="yes"

# limits for timeline cleanup
TIMELINE_MIN_AGE="1800"
TIMELINE_LIMIT_HOURLY="20"
TIMELINE_LIMIT_DAILY="20"
TIMELINE_LIMIT_WEEKLY="1"
TIMELINE_LIMIT_MONTHLY="0"
TIMELINE_LIMIT_YEARLY="0"


# cleanup empty pre-post-pairs
EMPTY_PRE_POST_CLEANUP="yes"

# limits for empty pre-post-pair cleanup
EMPTY_PRE_POST_MIN_AGE="1800"
```
</details>

# Firewall setup
Install `nftables` package:
```bash
pacman -S nftables
```
Edit the `/etc/nftables.conf` file to your needs:
1) Drop input and forward hooks
2) Allow all outbound and forwarded traffic
3) Allow established and related traffic
4) Allow SSH traffic
5) Allow ICMP traffic
6) Allow localhost (loopback) traffic
<details>
  <summary><i>Expand to see the config file</i></summary>

```
#!/usr/sbin/nft -f

flush ruleset

table inet filter {
	chain input {
		type filter hook input priority filter; policy drop;
		ct state invalid drop comment "early drop of invalid connections"
		ct state { established, related } accept comment "allow tracked connections"
		iifname "lo" accept comment "allow from loopback"
		ip protocol icmp accept comment "allow icmp"
		meta l4proto ipv6-icmp accept comment "allow icmp v6"
		tcp dport 22 accept comment "allow sshd"
	}

	chain forward {
		type filter hook forward priority filter; policy drop;
		ct state { established, related } accept
	}
}
```
</details>


Load the configuration:
```bash
nft -f /etc/nftables.conf
```
The configuration can be tested with:
```bash
nft list ruleset
```
Enable and start the `nftables` service:
```bash
systemctl enable --now nftables
```

# BIRD
Install the `bird` package:
```bash
pacman -S bird
```

Edit the `/etc/bird.conf` file to your needs to configure the BGP protocol:
1) Set the router ID
2) Enable the `device` protocol
3) Import and expoart all routes from the `direct` protocol and `kernel` protocol
4) Setup the `BGP` protocol
<details>
  <summary><i>Expand to see the config file</i></summary>

```conf

log syslog all;
router id 10.0.0.42;
protocol device {
}

protocol direct {
	ipv4;			# Connect to default IPv4 table
	ipv6;			# ... and to default IPv6 table
}

protocol kernel {
	ipv4 {			

	      import all;	# Import to table, default is import all
	      export all;	# Export to protocol. default is export none
	};

}

# Another instance for IPv6, skipping default options
protocol kernel {
	ipv6 { export all; };
}

protocol static {
	ipv4;			# Again, IPv4 channel with default options


}
protocol bgp {

  local 10.0.0.42 as 65042; # Local AS number
  neighbor 10.0.0.1 as 65001; # Remote AS number 
  multihop; # Allow multihop connections
  ipv4 {
    export all; # Export all routes to the neighbor
    import all; # Import all routes from the neighbor
    next hop self; # Use local address as nexthop for all routes
  };
};

```
</details>

Enable and start the `bird` service:
```bash
systemctl enable --now bird
```
The routing table can be displayed with:
```bash
birdc show route
```
Or more verbosely with:
```bash
birdc show route all
```
To view all protocols use:
```bash
sudo birdc show protocols
```



