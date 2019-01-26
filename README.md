## Installation

SSD Cell Cleaning can be using with [ThinkPad Drive Erase Utility](https://pcsupport.lenovo.com/tr/tr/downloads/ds019026)


**Define install disk, root&boot size, mirror**
```
export INSTALL_DISK="/dev/sda"
export ROOT_SIZE="32897"
export MIRROR_URL="http://ftp.linux.org.tr/gentoo/"
```

**Partitioning**
```
parted -a optimal $INSTALL_DISK --script "mklabel gpt"
parted -a optimal $INSTALL_DISK --script "unit mib"
parted -a optimal $INSTALL_DISK --script "mkpart ESP fat32 1 129"
parted -a optimal $INSTALL_DISK --script "set 1 boot on"
parted -a optimal $INSTALL_DISK --script "mkpart primary 129 $ROOT_SIZE"
parted -a optimal $INSTALL_DISK --script "name 2 root"
parted -a optimal $INSTALL_DISK --script "mkpart primary $ROOT_SIZE -1"
parted -a optimal $INSTALL_DISK --script "name 3 home"
export BOOT_PARTITION=${INSTALL_DISK}1
export ROOT_PARTITION=${INSTALL_DISK}2
export HOME_PARTITION=${INSTALL_DISK}3
```

**Creating File Systems**
```
mkfs.fat -F 32 $BOOT_PARTITION
mkfs.ext4 -F -E discard $ROOT_PARTITION
mkfs.ext4 -F -E discard $HOME_PARTITION
```

**Mounting**
```
mkdir -p /mnt/gentoo
mount -o discard $ROOT_PARTITION /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount $BOOT_PARTITION /mnt/gentoo/boot
mkdir -p /mnt/gentoo/boot/EFI
mkdir -p /mnt/gentoo/boot/EFI/gentoo
mkdir -p /mnt/gentoo/home
mount -o discard $HOME_PARTITION /mnt/gentoo/home
```


**Downloading stage**
```
export LASTEST_STAGE=$(curl -s ${MIRROR_URL}releases/amd64/autobuilds/latest-stage3-amd64.txt | tail -n 1 | cut -d " " -f 1)
wget ${MIRROR_URL}releases/amd64/autobuilds/$LASTEST_STAGE -O /tmp/latest-stage3-amd64.tar.bz2
```

**Unpacking stage**
```
tar xpvf /tmp/latest-stage3-amd64.tar.bz2 --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo/ && sync
rm -f /tmp/latest-stage3-amd64.tar.bz2
```

**Configs**
```
nano -w /mnt/gentoo/etc/portage/make.conf

```

**Chroot**
```
cp -Lrf /etc/resolv.conf /mnt/gentoo/etc/
mount -t proc /proc /mnt/gentoo/proc
mount --rbind /sys /mnt/gentoo/sys
mount --make-rslave /mnt/gentoo/sys
mount --rbind /dev /mnt/gentoo/dev
mount --make-rslave /mnt/gentoo/dev
mount --rbind /tmp /mnt/gentoo/tmp
chroot /mnt/gentoo /bin/bash
source /etc/profile
export PS1="(chroot) $PS1"
```

**Init portage**
```
emerge-webrsync
emerge --sync
eselect news list 
eselect news read
```

**World**
```
emerge --ask --update --deep --newuse @world -j2
emerge --ask --depclean 
```

**FSTAB**
```
BOOT_PARTUUID=$(blkid $BOOT_PARTITION | cut -d '"' -f 8)
ROOT_PARTUUID=$(blkid $ROOT_PARTITION | cut -d '"' -f 8)
HOME_PARTUUID=$(blkid $HOME_PARTITION | cut -d '"' -f 8)
# and write template
```

**Kernel**
```
emerge --oneshot sys-kernel/ck-sources sys-kernel/linux-firmware sys-apps/pciutils sys-apps/usbutils -j 4
#copy ck config
#...

make -j8
make modules_install
make install
cp arch/x86/boot/bzImage /boot/EFI/gentoo/bzImage.efi
```

**Bootloader**
```
emerge --ask sys-boot/efibootmgr
mount /sys/firmware/efi/efivars -o rw,remount
efibootmgr -c -d /dev/sda -p 1 -L "gentoo" -l "\efi\gentoo\bzImage.efi"
mount /sys/firmware/efi/efivars -o ro,remount
```

**MICS**
```
/etc/conf.d/hostname
passwd
nano /etc/conf.d/keymaps

useradd -m -G audio,cdrom,portage,usb,users,video,wheel,audio -s /bin/bash hexvalid
passwd hexvalid

emerge --ask net-misc/dhcpcd
emerge --ask --noreplace net-misc/netifrc
emerge --ask --changed-use net-misc/openssh
# sudo 

```




**X**
```
emerge --ask --verbose x11-base/xorg-drivers x11-base/xorg-server x11-apps/setxkbmap x11-apps/xrandr x11-apps/xinit -j2
emerge --ask x11-wm/i3-gaps x11-misc/i3blocks x11-terms/rxvt-unicode x11-misc/dmenu media-gfx/feh -j 3
```
