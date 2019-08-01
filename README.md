## Installation

**1.1 NVMe SSD Cell Cleaning**
```
nvme format -s1 /dev/nvme0n1
```

****Define target install disk, root and boot partition size, and portage mirror**

For my X250's SSD:
```
export INSTALL_DISK="/dev/sda"
export BOOT_SIZE="128"
export ROOT_SIZE="32768"
export MKFS_PARAMS="-F -i 16384 -E discard"
export MOUNT_PARAMS="-o discard"
```

or sometimes i using my [Sandisk Ultra Fit 32GB](https://www.sandisk.com.tr/home/usb-flash/ultra-fit-usb) USB flash disk for testing purposes:
```
export INSTALL_DISK="/dev/sdb"
export BOOT_SIZE="64"
export ROOT_SIZE="16384"
export MKFS_PARAMS="-F -i 4096 -T small"
export MOUNT_PARAMS=""
```

and portage mirror closest to me:
```
export MIRROR_URL="http://ftp.linux.org.tr/gentoo/"
```

**Partitioning**
```
parted -a optimal $INSTALL_DISK --script "mklabel gpt"
parted -a optimal $INSTALL_DISK --script "unit mib"
parted -a optimal $INSTALL_DISK --script "mkpart ESP fat32 1 $(($BOOT_SIZE+1))"
parted -a optimal $INSTALL_DISK --script "set 1 boot on"
parted -a optimal $INSTALL_DISK --script "mkpart primary $(($BOOT_SIZE+1)) $((ROOT_SIZE+$BOOT_SIZE+1))"
parted -a optimal $INSTALL_DISK --script "name 2 root"
parted -a optimal $INSTALL_DISK --script "mkpart primary $((ROOT_SIZE+$BOOT_SIZE+1)) -1"
parted -a optimal $INSTALL_DISK --script "name 3 home"
```

****Define partitions**
```
export BOOT_PARTITION=${INSTALL_DISK}1
export ROOT_PARTITION=${INSTALL_DISK}2
export HOME_PARTITION=${INSTALL_DISK}3
```

**Creating File Systems**
```
mkfs.fat -F 32 $BOOT_PARTITION
mkfs.ext4 $MKFS_PARAMS $ROOT_PARTITION
mkfs.ext4 $MKFS_PARAMS $HOME_PARTITION
```

****Mount partitions**
```
mkdir -p /mnt/gentoo
mount $MOUNT_PARAMS $ROOT_PARTITION /mnt/gentoo
mkdir -p /mnt/gentoo/boot
mount $BOOT_PARTITION /mnt/gentoo/boot
mkdir -p /mnt/gentoo/boot/EFI
mkdir -p /mnt/gentoo/boot/EFI/gentoo
mkdir -p /mnt/gentoo/home
mount $MOUNT_PARAMS $HOME_PARTITION /mnt/gentoo/home
```

**Downloading latest stage**
```
export LATEST_STAGE=$(curl -s ${MIRROR_URL}releases/amd64/autobuilds/latest-stage3-amd64.txt | tail -n 1 | cut -d " " -f 1)
wget ${MIRROR_URL}releases/amd64/autobuilds/$LATEST_STAGE -O /tmp/latest-stage3-amd64.tar.bz2
```

**Unpacking stage and clean**
```
tar xpvf /tmp/latest-stage3-amd64.tar.bz2 --xattrs-include='*.*' --numeric-owner -C /mnt/gentoo/ && sync
for i in `find /mnt/gentoo/ -name .keep`; do rm $i; done
rm -f /tmp/latest-stage3-amd64.tar.bz2 #if you want
```

**Portage Config**
*When this file final, this section will be change.*
```
rm -rf /mnt/gentoo/etc/portage/package.use
# nano -w /mnt/gentoo/etc/portage/package.use
sys-devel/gcc nls

```


```
# nano -w /mnt/gentoo/etc/portage/make.conf
COMMON_FLAGS="-march=broadwell -mabm -O2 -pipe"
CFLAGS="${COMMON_FLAGS}"
CXXFLAGS="${COMMON_FLAGS}"
FCFLAGS="${COMMON_FLAGS}"
FFLAGS="${COMMON_FLAGS}"
CHOST="x86_64-pc-linux-gnu"
CPU_FLAGS_X86="aes avx avx2 f16c fma3 mmx mmxext pclmul popcnt sse sse2 sse3 sse4_1 sse4_2 ssse3"
MAKEOPTS="-j3"
PORTDIR="/usr/portage"
DISTDIR="/usr/portage/distfiles"
PKGDIR="/usr/portage/packages"
PORTAGE_TMPDIR="/tmp/"
KDIR=/usr/src/linux
GENTOO_MIRRORS="http://ftp.linux.org.tr/gentoo/"
LC_MESSAGES=C
LINGUAS="" 
L10N="" 
FEATURES="noman"
INPUT_DEVICES="keyboard mouse libinput synaptics"
VIDEO_CARDS="intel i965"
USE="minimal -man -doc -nls -ipv6 unicode threads"
```

****Chroot**
```
cp --dereference /etc/resolv.conf /mnt/gentoo/etc/
mount --types proc /proc /mnt/gentoo/proc
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
emerge --oneshot sys-kernel/gentoo-sources sys-kernel/linux-firmware sys-apps/pciutils sys-apps/usbutils -j 4
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

useradd -m -G audio,cdrom,portage,usb,users,video,wheel -s /bin/bash hexvalid
passwd hexvalid

emerge --ask net-misc/dhcpcd
emerge --ask --noreplace net-misc/netifrc
emerge --ask --changed-use net-misc/openssh
# REQ PACKAGES: app-admin/sudo dev-vcs/git www-client/firefox app-shells/bash-completion


```




**X**
```
emerge --ask --verbose x11-base/xorg-drivers x11-base/xorg-server x11-apps/setxkbmap x11-apps/xrandr x11-apps/xinit -j2
emerge --ask x11-wm/i3-gaps x11-misc/i3blocks x11-terms/rxvt-unicode x11-misc/dmenu media-gfx/feh -j 3
```
