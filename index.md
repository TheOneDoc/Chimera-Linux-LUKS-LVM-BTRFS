# Chimera Linux

## Goal

This document will walk you through installing [Chimera Linux](https://en.wikipedia.org/wiki/Chimera_Linux)
with [KDE Plasma](https://kde.org/plasma-desktop/) on top of a [Snapper](http://snapper.io/) ready [__BTRFS__](https://en.wikipedia.org/wiki/Btrfs) file system
that resides inside a [Logical Volume Manager](https://en.wikipedia.org/wiki/Logical_Volume_Manager_(Linux)) (__LVM__) Logical Volume (__LV__) inside a Volume Group (__VG__) 
inside a encrypted [Linux Unified Key Setup](https://en.wikipedia.org/wiki/Linux_Unified_Key_Setup) (__LUKS__) partition

Note: The configuration and usage of Snapper is out of scope for this document.

## Target System

In this example our target system is a GMKtec NucBox K11 Mini-PC running 
- CPU AMD Ryzen™ 9 8945HS 8 Cores 16 Threads 
- Ram 32 GB
- Drive 2TB
- booted via [Unified Extensible Firmware Interface](https://en.wikipedia.org/wiki/UEFI) (UEFI)
- System Language EN/DE

You have to adjust accordingly for your own needs.
Please utilize the official [Chimera Linux Documentation](https://chimera-linux.org/docs/))


## Preparation


the current official Chimera Linux [Plasma KDE live Image](https://repo.chimera-linux.org/live/latest/) is our installation environment.

Please adjust the following URL accordingly as the one in this guide is current as of writing.
```
https://repo.chimera-linux.org/live/latest/chimera-linux-x86_64-LIVE-20251220-plasma.iso
```
For validation of the official image see [Chimera Linux Installation Documentation](https://chimera-linux.org/docs/installation)

Important: I provide more frequently updated Chimera Live Images for AMD64/x86_64 and AARCH64 [here](https://c.1und1.de/@1632165589407503469/fVfz_5G2xzpIEFlMtFCZVQ)

This images come with aditional packages pre-installed and this guide assumes that my plasmakvm image is used

Please see [This Fediverse post](https://tech.lgbt/@TheOneDoc/116505561647485255) for the current state of the images.

Now it's time to boot our machine/VM from the installation environment.

After the system is booted we start with the actual Installation.

## Installation

### What to install?

Bootstrapping Chimera Linux is very simmilar to bootsrapping ARCH or Debian

### Prepare the Installation Environment

Chimera Linux defaults to the [__doas__](https://en.wikipedia.org/wiki/Doas) command to execute tasks with superuser (root) privileges.

#### Open the __Console__ by starting [__Konsole__](https://en.wikipedia.org/wiki/Konsole)

#### Change root password and create our install user
Chimera Linux Live Images come pre-configured wth two useres.

the ```root``` user with the default password ```chimera```
and the ```anon``` user with the default password ```chimera```.

```anon``` is a member of the group ```wheel``` and can perform tasks as superuser via the ```doas``` command

```
doas -s
passwd
useradd -m -G users,wheel uwe
passwd uwe
su - uwe
```
![](0001.png)

#### (optional) start sshd
This step can be skipped if the installation is done locally
```
doas dinitctl enable sshd
```
#### (optional) get the IP Address of our Install Environment
```
ip a
```
![](0002.png)

At this point you can either continue the installation locally or ssh into the Installation Environment and continue from remote 

All the following steps need to be performed with superuser (root) privileges

```
doas -s
```
![](0003.png)

### (optional) update the installation environment
If you use the official Chimera Live Image you now have to update, make the __users__ repository available and install some tools

```
doas -s
apk update
apk upgrade --latest
apk add chimera-repo-user
apk update
apk upgrade --latest
apk add parted nvme-cli tmux
```
![](0003-1.png)

### prepare Installation target

our target device is called __/dev/nvme0n1__ because it's a [NVMe Express m.2 SSD](https://en.wikipedia.org/wiki/NVM_Express) [block device](https://en.wikipedia.org/wiki/Device_file#Block_devices) other valid targets are
```
/dev/vd?
/dev/sd?
/dev/nvme?n?
/dev/mmcblk?
```
Note: replace ? with the number/letter that specifies your target device

#### sanitize the target device

##### Wipe the device
Remove all File system, RAID and Partition Table signatures from target device
```
wipefs -a /dev/nvme0n1
```

##### (optional) secure erase the target drive

For NVME SSDs perform ```nvme sanitize /dev/nvme0 -a 0x02``` and for all other block devices perform ```blkdiscard -vfz /dev/nvme0n1```.

Note: This operation can take a long time depending on size and speed of the target drive.

#### Partitioning the target device

##### Check that the target is indeed empty
```
parted /dev/nvme0n1 print
```
![](0004.png)

##### Create the Partition Table and Partitions
```
parted /dev/nvme0n1 mklabel gpt
parted /dev/nvme0n1 mkpart primary fat32 0% 2.5GB
parted /dev/nvme0n1 name 1 esp
parted /dev/nvme0n1 set 1 esp on
parted /dev/nvme0n1 set 1 boot on
parted /dev/nvme0n1 mkpart primary 2.5GB 100%
parted /dev/nvme0n1 name 2 LUKS-crypt
```

##### Check that the target device is correctly partitioned
```
parted /dev/nvme0n1 print
```
![](0005.png)

#### EFI File system creation
```
mkfs.fat -F 32 -n EFI /dev/nvme0n1p1
```

#### LUKS Partition creation
```
cryptsetup luksFormat /dev/nvme0n1p2
```
unlock the LUKS Container and map it to __/dev/mapper/crypt__
```
cryptsetup luksOpen /dev/nvme0n1p2 crypt
```
(optional) enable discards on __LUKS__

SSD/SD/MMC block devices should enable discards
```
cryptsetup refresh --persistent --allow-discards crypt
```
![](0006.png)

#### LVM configuration

Create a Physical Volume (__PV__) mapped to  our __LUKS__ Container and a Volume Group (__VG__) named __system__ 
Note: the Physical Volume (__PV__) creation on LUKS container is implicit.
```
vgcreate system /dev/mapper/crypt
```
We create two Logical Volumes (__LV__) in our Volume Group (__VG__) __system__. 

The first __LV__ contains our swap. We name it __swap__.

A good size for it is RAM Size * 2.5
```
lvcreate --name swap -L 80G system
```
The Second __LV__ contains our __BTRFS__ File System. We name it __root__. 
```
lvcreate --name root -l 100%free system
```
The __LVs__ are mapped into the system as ```/dev/{VG}/{LV}``` and ```/dev/mapper/{VG}-{LV}```
![](0007.png)

#### File system Creation

Create swap on ```/dev/system/swap``` and name it __swapfs__
```
mkswap /dev/system/swap -L swapfs
```
Enable swap
```
swapon /dev/system/swap
```
![](0008.png)

Create BTRFS on ``/dev/mapper/system-root`` and name it __rootfs__
```
mkfs.btrfs -f -L rootfs /dev/mapper/system-root
```
![](0009.png)

Create the BTRFS Subvolumes
```
mount -v -m -t btrfs -o ssd,compress=zstd:11,subvol=/ /dev/system/root /media/root
btrfs subvolume create /media/root/@
btrfs subvolume create /media/root/@home
btrfs subvolume create /media/root/@root
btrfs subvolume create /media/root/@var@log
btrfs subvolume create /media/root/@snapshots
btrfs subvolume create /media/root/@home/.snapshots
```
![](0010.png)

check if all is done correctly
```
btrfs subvolume list /media/root
```
![](0011.png)

Set __@__ as the __default__ subvolume
```
btrfs subvolume set-default /media/root/@
```
check if it's set correctly
```
btrfs subvolume get-default /media/root
```
![](0012.png)
cleanup
```
umount /media/root
```
#### Mount File systems for use in the chroot Environment
```
mount -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@ /dev/system/root /media/root
chmod 755 /media/root
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@root /dev/system/root /media/root/root
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@var@log /dev/system/root /media/root/var/log
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@snapshots /dev/system/root /media/root/.snapshots
mount -m -t btrfs -o compress=zstd:11,ssd,noatime,subvol=/@home /dev/system/root /media/root/home
mount -m -t vfat /dev/nvme0n1p1 /media/root/boot
```
check if everything is mounted in the right place
```
findmnt -R /media/root
```
![](0013.png)

### Bootstrap Chimera Linux

Chimera Linux's ```chimera-bootstrap``` per default installs the ```base-full``` package from the official repository into the traget directory.

This is the safest option and the only one covered in this guide.

```
cd
chimera-bootstrap /media/root
```
![](0014.png)

### chroot into the new Chimera Linux Installation

```
doas chimera-chroot /media/root
export PS1="(chroot) \u@\h:\W # "
```
![](0015.png)


#### Check mounts

```
lsblk -o NAME,FSTYPE,UUID,PARTUUID,RO,RM,SIZE,STATE,OWNER,GROUP,MODE,TYPE,MOUNTPOINT,LABEL,MODEL
findmnt -R /
```
Note: UUIDs device names and size can differ.
![](0016.png)


#### Update Chimera Linux

##### (Optional) Set the path to a chimera mirror

If no Mirror is specified the default Repository will be used

```
mkdir -p /etc/apk/repositories.d
```

```
cat << 'EOF' > /etc/apk/repositories.d/00-my-local-repo.list
#This is the default
#set CHIMERA_REPO_URL=https://repo.chimera-linux.org
#Path to the Chimera Repo local mirror directory on this machine
set CHIMERA_REPO_URL=http://chimeramirror.fritz.box
EOF
```
##### Update the Installation and configure the users Repository

In case of errors run ```apk fix```.

```
apk update
apk upgrade
apk add chimera-repo-user
apk update
apk upgrade --available
```
![](0017.png)


##### User Configuration

Set root user account password
```
passwd root
```
Create main user account and set password
```
useradd -m -G wheel,kvm,users,plugdev -s /bin/sh uwe
passwd uwe
```

##### Set the hostname
```
echo myhostname > /etc/hostname
```

##### Set the Timezone
The Timezone is set to CET

Adjust accordingly.
```
ln -sf /usr/share/zoneinfo/Europe/Berlin /etc/localtime
```
##### Configure keymaps

```
cat << 'EOF' > /etc/default/keyboard
# KEYBOARD CONFIGURATION FILE

# Consult the keyboard(5) manual page.

KMAP=de

XKBMODEL=pc105
XKBLAYOUT=de
XKBVARIANT=nodeadkeys
XKBOPTIONS=

BACKSPACE=guess
EOF
```

Now is a good moment to read up on Chimera Linux' [Package Management](https://chimera-linux.org/docs/apk) and [Service Management](https://chimera-linux.org/docs/configuration/services)


This is a quick one-liner to install all needed packages.
Please look at the individual sections for details
```
apk add linux-lts linux-latest grub-x86_64-efi plasma-desktop flatpak smartmontools ufw firefox thunderbird networkmanager bash bash-completion fish-shell qemu-guest-agent-dinit spice-vdagent-dinit
```


##### Install NetworkManager
We use Network Manager for Network device configuration
```
apk add networkmanager
dinitctl enable -o networkmanager
```

##### Enable syslog-ng
The default logging system on Chimera is syslog-ng, which is part of base-full. Enable the syslog daemon as follows

```
dinitctl enable -o syslog-ng
```

##### Enable sshd

```
dinitctl enable -o sshd
```

##### Install File indexing

```
emerge --ask sys-apps/mlocate
```

##### Install bash completion

```
emerge --ask app-shells/bash-completion
```

##### Install gentoolkit

```
emerge --ask app-portage/gentoolkit
```

##### Install File Sytem tools

```
emerge --ask sys-fs/cryptsetup
emerge --ask sys-fs/btrfs-progs
emerge --ask sys-fs/dosfstools
emerge --ask sys-fs/xfsprogs
emerge --ask sys-fs/e2fsprogs
emerge --ask sys-fs/ntfs3g
emerge --ask sys-fs/f2fs-tools
emerge --ask sys-fs/mdadm
emerge --ask dev-python/zstandard
emerge --ask sys-block/io-scheduler-udev-rules
```

##### Install the Logical Volume Manager

Set the USAGE flag
```
cat << 'EOF' >> /etc/portage/package.use/lvm2
#Enable support for the LVM daemon and related tools
sys-fs/lvm2 lvm
EOF
```
Install and enable LVM
```
emerge --ask sys-fs/lvm2
rc-update add lvm boot
```

##### Build /etc/fstab
```
emerge --ask sys-fs/genfstab
genfstab -U / >> /etc/fstab
```
check generated file
```
cat /etc/fstab
```
![](0021.png)

##### Build ```/etc/crypttab```
Get the __UUID__ for the LUKS Container on ```/dev/nvme0n1p2```

```
lsblk -o NAME,FSTYPE,UUID,TYPE,MOUNTPOINT,LABEL
```
![](0022.png)

Write ```/etc/crypttab```

```
echo "crypt UUID=1e5b53d9-03d8-4545-803e-3cc69eeac52d none luks,discard" > /etc/crypttab
```
##### Install Linux Firmware
```
emerge --ask sys-kernel/linux-firmware
emerge --ask sys-firmware/sof-firmware
```
##### Install the Linux Kernel
Make sure that __ugrd__ and __grub__ are used by the kernel

```
cat << 'EOF' >> /etc/portage/package.use/installkernel
sys-kernel/installkernel -dracut ugrd grub
EOF
```
Make sure that Grub uses the EFI
```
echo 'GRUB_PLATFORMS="efi-64"' >> /etc/portage/make.conf
```
make sure that external kmods get auto rebuild
```
echo 'USE="${USE} dist-kernel"' >> /etc/portage/make.conf
```

Misc config (not needed?)

```
mkdir -p /etc/kernel
echo "quiet splash" > /etc/kernel/cmdline
mkdir -p /etc/cmdline.d
ln -s /etc/kernel/cmdline /etc/cmdline.d/00-installkernel.conf
```

Install the gentoo default kernel binary
```
emerge --ask sys-kernel/gentoo-kernel-bin
```

Installkernel can be re-emerged and will pull UGRD

```
emerge -ask sys-kernel/installkernel
```

##### Install ZFS support

```
emerge --ask sys-fs/zfs
```
To force an initramfs rebuild, emerge --config can be used on dist-kernel packages:
```
emerge --config sys-kernel/gentoo-kernel-bin
```
##### Install GRUB
```
emerge --ask sys-boot/grub
grub-install --efi-directory=/boot
grub-mkconfig -o /boot/grub/grub.cfg
emerge sys-kernel/installkernel[grub]
```

##### Install some tools
```
emerge dev-vcs/git
emerge sys-process/btop
emerge app-misc/fastfetch
emerge app-misc/tmux
```

##### (optional) QEMU/KVM VM guest agent services
```
emerge --ask app-emulation/spice-vdagent
emerge --ask app-emulation/qemu-guest-agent
rc-update add spice-vdagent default
rc-service spice-vdagent start
rc-update add qemu-guest-agent default
rc-service qemu-guest-agent start
```

##### Install KDE
```
emerge --ask kde-plasma/plasma-meta kde-apps/kde-apps-meta
emerge --ask gui-libs/display-manager-init
```

##### Update __@world__ set
```
emerge --ask --verbose --update --deep --changed-use @world
```
###### enable __elogind__
```
rc-update add elogind boot
```
##### Install __SDDM__

```
emerge --ask x11-misc/sddm
emerge --ask kde-plasma/sddm-kcm

usermod -a -G video sddm
cat << 'EOF' > /etc/conf.d/display-manager
CHECKVT=7
DISPLAYMANAGER="sddm"
EOF

cat << 'EOF' > /etc/sddm.conf
[Users]
MaximumUid=60000
MinimumUid=1000
EOF

rc-update add display-manager default
rc-service display-manager start
```

##### Enable __seatd__
```
echo "sys-auth/seatd server" > /etc/portage/package.use/seatd
emerge sys-auth/seatd
rc-update add seatd default
rc-service seatd start
usermod -aG seat sddm
usermod -aG seat uwe
```


##### Cleanup
```
eclean-dist
eclean-pkg
rm /stage3-*.tar.*
```
All done time to exit the chrot
```
exit
````

And reboot the machine
```
sudo reboot
```
![The End Result](0023.png)
