# Install host system

The system is loaded with SeaBios.

## Configure the network

### Find out the name of your wireless network interface

```
sh> ls /sys/class/net
lo wlp1s0
```

`wlp1s0` is a wireless network interface, so we can be sure that the device driver was loaded successfully.

### Enable the interface

Check the status of the interface `wlp1s0`.

```
sh> ip link show dev wlp1s0
2: wlp1s0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
  link/ether 6c:29:95:a6:85:97 brd ff:ff:ff:ff:ff:ff
```

The interface is disabled because `<BROADCAST,MULTICAST>` misses `UP` in it.

Enable the interface.

```
sh> ip link set wlp1s0 up
sh> ip link show dev wlp1s0
2: wlp1s0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
  link/ether 6c:29:95:a6:85:97 brd ff:ff:ff:ff:ff:ff
```

### Connect to the network

We'll be using `wpa_supplicant (8)` (http://w1.fi/wpa_supplicant/) to connect to the Wi-Fi network. `wpa_supplicant` is Wi-Fi Protected Access client and IEEE 802.1X supplicant.

```
sh> vi /etc/wpa_supplicant/wpa_supplicant.conf
sh> cat /etc/wpa_supplicant/wpa_supplicant.conf
ctrl_interface=/run/wpa_supplicant
update_config=1
```

In a different tty start `wpa_supplicant` daemon with debugging enabled.

```
sh> wpa_supplicant -iwlp1s0 -c/etc/wpa_supplicant/wpa_supplicant.conf -d
```

Get back to original tty and configure Access Point connection with `wpa_cli`.

```
sh> wpa_cli
> add_network
0
> set_network 0 ssid "MYSSID"
OK
> set_network 0 psk "passphrase"
OK
> enable_network 0
OK
> save_config
OK
> quit
```

Switch back to the tty, where `wpa_suppicant` is running.
The configuration will be hot-reloaded which should be reflected in the log output.
A series of state transitions will be reported.
If successful, the connection will be established.

```
sh> ip link show dev wlp1s0
2: wlp1s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000
  link/ether 6c:29:95:a6:85:97 brd ff:ff:ff:ff:ff:ff
```

Obtain an IP address using `dhcpcd`.

```
sh> dhcpcd wlp1s0
[...]
sh> ping www.google.com
[...]
```

## Update the system clock

```
sh> timedatectl set-ntp true
sh> timedatectl status
                      Local time: Sun 2018-07-29 03:46:55 UTC
                  Universal time: Sun 2018-07-29 03:46:55 UTC
                        RTC time: Sun 2018-07-29 03:46:55
                       Time zone: UTC (UTC, +0000)
       System clock synchronized: yes
systemd-timesyncd.service active: yes
                 RTC in local TZ: no
```

## Partition the disk and setup file systems

```
sh> lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0    7:0    0  463M  1 loop /run/archiso/sfs/airootfs
sda      8:0    0 14.9G  0 disk 
├─sda1   8:1    0   11G  0 part 
├─sda2   8:2    0    1K  0 part 
└─sda5   8:5    0    4G  0 part 
sdb      8:16   1 14.8G  0 disk 
├─sdb1   8:17   1  574M  0 part /run/archiso/bootmnt
└─sdb2   8:18   1   64M  0 part 
sh> fdisk /dev/sda
sh> lsblk
```

Create filesystems.

```
sh> mkfs.ext4 /dev/sda1 # for host system
sh> mkfs.ext4 /dev/sda2 # for Vanilla Linux system
sh> mkswap /dev/sda3
sh> swapon /dev/sda3
```

Mount the to-be-root partition.

```
sh> mount /dev/sda1 /mnt
```

## Install and configure the system

```
sh> pacstrap /mnt base
```

Generate fstab file.

```
sh> genfstab -U /mnt >> /mnt/etc/fstab
```

Chroot and configure

```
sh> arch-chroot /mnt
sh> pacman -S vim
sh> pacman -S tmux
sh> pacman -S wpa_supplicant
sh> ln -sf /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
sh> hwclock --systohc
```

Uncomment `en_US.UTF-8 UTF-8` in /etc/locale.gen

```
sh> locale-gen
sh> vi /etc/locale.conf
sh> cat /etc/locale.conf
LANG=en_US.UTF-8
```

Configure network.

```
sh> vi /etc/hostname
sh> cat /etc/hostname
myhostname
sh> vi /etc/hosts
sh> cat /etc/hosts
127.0.0.1      localhost
::1            localhost
127.0.0.1      myhostname.localdomain myhostname
```

Set the root password

```
sh> passwd
```

## Install boot loader

In order to boot Arch Linux, you must install a boot loader to the Master Boot Record.
The boot loader is the first piece of software started by SeaBios.
Boot loaders only need to support the file system on which kernel and initramfs reside (the file system on which `/boot` is located).
It is responsible for loading the kernel with the wanted kernel parameters and initial RAM disk before initiating the boot process.

```
sh> pacman -S grub
sh> grub-install --target=i386-pc /dev/sda
sh> grub-mkconfig -o /boot/grub/grub.cfg
```

## Wrapping up

Install intel-ucode and enable microcode updates.

Exit chroot and unmount all the partitions with `umount -R /mnt`.

```
sh> reboot
```

## Post-installation

Create user

```
sh> useradd --create-home alex
sh> passwd alex
```

Install GUI

```
sh> pacman -S xorg-server
sh> pacman -S xorg-xinit
sh> pacman -S xterm
sh> pacman -S i3 # be sure to select i3-wm, not i3-gaps as the latter doesn't work.
sh> pacman -S chromium
```

Identify video card

```
sh> lspci | grep -e VGA -e 3D
00:02.0 VGA compatible controller: Intel Corporation HD Graphics (rev 08)
sh> pacman -S xf86-video-intel
```
