# LFS - ft_linux project of school42

## 1. Check that all system requirements are fulfilled with the [script](resources/version-check.sh)


## 2. Preparing the Host System
We'll need a new empty partition of 30 Gb. I used an iSCSI lun from storage system and performed the following steps:
* create a partition table (gpt). A partition table is located at the start of a hard drive and it stores data about the size and location of each partition.
```
sudo parted /dev/sda
> mklabel gpt
```
* create boot, root, swap and BIOS-boot partitions, arguments are partition's type/filesystem/start/end
```
mkpart primary ext4 20MB 230MB
mkpart primary ext4 230MB 27GB
mkpart swap linux-swap 27GB 100%
mkpart primary ext4 1MB 20MB
```
* format the primary partitions
```
sudo mkfs.ext4 /dev/sda1
sudo mkfs.ext4 /dev/sda2
```
* initialize swap partition
```
sudo mkswap /dev/sda3
```
* change boot partition type to BIOS boot
```
sudo fdisk /dev/sda

Command (m for help): t
Partition number (1-3, default 3): 4
Partition type (type L to list all types): L
...
4 BIOS boot                      21686148-6449-6E6F-744E-656564454649
...

Partition type (type L to list all types): 4

Changed type of partition 'Linux filesystem' to 'BIOS boot'.

Command (m for help): w
```
* edit .bashrc file of current user and root user adding the following line
```
export LFS=/mnt/lfs
```
* create the mount point and mount the LFS file system by running:
```
mkdir -pv $LFS
mount -v -t ext4 /dev/sda2 $LFS
swapon -s # check that the swap partition is recognized
```

## 3. Packages and Patches
* create sources directory and change permissions to allow writing with a sticky bit, that restricts the deletion of the file to make it possible only for the owner
```
mkdir -v $LFS/sources
chmod -v a+wt $LFS/sources
```
* create in current directory [wget-list file](resources/wget-list) and run the following command to download all the packages
```
wget --input-file=wget-list --continue --directory-prefix=$LFS/sources
```

## 4. Final preparations
* create directory layout
```
sudo mkdir -pv $LFS/{etc,var} $LFS/usr/{bin,lib,sbin}

for i in bin lib sbin; do
  sudo ln -sv usr/$i $LFS/$i
done

case $(uname -m) in
  x86_64) sudo mkdir -pv $LFS/lib64 ;;
esac

sudo mkdir -pv $LFS/tools
```
* add LFS user
```
groupadd lfs
useradd -s /bin/bash -g lfs -m -k /dev/null lfs
passwd lfs
```
* make lfs user the owner of the directory
```
chown -v lfs $LFS/{usr{,/*},lib,var,etc,bin,sbin,tools}
case $(uname -m) in
  x86_64) chown -v lfs $LFS/lib64 ;;
esac
```
* login as lfs user:
```
su - lfs
```
* create .bash_profile and .bashrc files and make a source the first one
```
cat > ~/.bash_profile << "EOF"
exec env -i HOME=$HOME TERM=$TERM PS1='\u:\w\$ ' /bin/bash
EOF

cat > ~/.bashrc << "EOF"
set +h
umask 022
LFS=/mnt/lfs
LC_ALL=POSIX
LFS_TGT=$(uname -m)-lfs-linux-gnu
PATH=/usr/bin
if [ ! -L /bin ]; then PATH=/bin:$PATH; fi
PATH=$LFS/tools/bin:$PATH
CONFIG_SITE=$LFS/usr/share/config.site
export LFS LC_ALL LFS_TGT PATH CONFIG_SITE
EOF

source ~/.bash_profile
```
* from root run the following command to remove /etc/bash.bashrc file from the way
```
[ ! -e /etc/bash.bashrc ] || mv -v /etc/bash.bashrc /etc/bash.bashrc.NOUSE
```

* finally check the host system so that:
```
/usr/bin/awk is a symbolic link to gawk.
/usr/bin/yacc is a symbolic link to bison or a small script that executes bison.
```

## 5. Compiling a Cross-Toolchain
* extract binutils tar file and create build directory inside extracted directory, set options for compilation and compile, remove directory
```
tar -xf binutils-2.38.tar.xz
cd binutils-2.38
mkdir -v build
cd       build
../configure --prefix=$LFS/tools \
             --with-sysroot=$LFS \
             --target=$LFS_TGT   \
             --disable-nls       \
             --disable-werror
make
make install
cd ../..
rm -rf binutils-2.38
```
* compile gcc
```
tar -xf gcc-11.2.0.tar.xz
cd gcc-11.2.0
tar -xf ../mpfr-4.1.0.tar.xz
mv -v mpfr-4.1.0 mpfr
tar -xf ../gmp-6.2.1.tar.xz
mv -v gmp-6.2.1 gmp
tar -xf ../mpc-1.2.1.tar.gz
mv -v mpc-1.2.1 mpc

case $(uname -m) in
  x86_64)
    sed -e '/m64=/s/lib64/lib/' \
        -i.orig gcc/config/i386/t-linux64
 ;;
esac

mkdir -v build
cd       build

../configure                  \
    --target=$LFS_TGT         \
    --prefix=$LFS/tools       \
    --with-glibc-version=2.35 \
    --with-sysroot=$LFS       \
    --with-newlib             \
    --without-headers         \
    --enable-initfini-array   \
    --disable-nls             \
    --disable-shared          \
    --disable-multilib        \
    --disable-decimal-float   \
    --disable-threads         \
    --disable-libatomic       \
    --disable-libgomp         \
    --disable-libquadmath     \
    --disable-libssp          \
    --disable-libvtv          \
    --disable-libstdcxx       \
    --enable-languages=c,c++

make
make install

cd ..
cat gcc/limitx.h gcc/glimits.h gcc/limity.h > \
  `dirname $($LFS_TGT-gcc -print-libgcc-file-name)`/install-tools/include/limits.h
cd ..
rm -rf gcc-11.2.0
```
* compile linux
```
tar -xf linux-5.16.9.tar.xz
cd linux-5.16.9
make mrproper
make headers
find usr/include -name '.*' -delete
rm usr/include/Makefile
cp -rv usr/include $LFS/usr
cd ..
rm -rf linux-5.16.9
```
* compile glibc
```
tar -xf glibc-2.35.tar.xz
cd glibc-2.35
case $(uname -m) in
    i?86)   ln -sfv ld-linux.so.2 $LFS/lib/ld-lsb.so.3
    ;;
    x86_64) ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64
            ln -sfv ../lib/ld-linux-x86-64.so.2 $LFS/lib64/ld-lsb-x86-64.so.3
    ;;
esac
patch -Np1 -i ../glibc-2.35-fhs-1.patch
mkdir -v build
cd       build
echo "rootsbindir=/usr/sbin" > configparms
../configure                             \
      --prefix=/usr                      \
      --host=$LFS_TGT                    \
      --build=$(../scripts/config.guess) \
      --enable-kernel=3.2                \
      --with-headers=$LFS/usr/include    \
      libc_cv_slibdir=/usr/lib
make
make DESTDIR=$LFS install
sed '/RTLDLIST=/s@/usr@@g' -i $LFS/usr/bin/ldd
```
run the following
```
echo 'int main(){}' > dummy.c
$LFS_TGT-gcc dummy.c
readelf -l a.out | grep '/ld-linux'
```
and it should result in the following: `[Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]`
continue
```
rm -v dummy.c a.out
$LFS/tools/libexec/gcc/$LFS_TGT/11.2.0/install-tools/mkheaders
cd ../..
rm -rf glibc-2.35
```
* compile gcc
```
tar -xf gcc-11.2.0.tar.xz 
cd gcc-11.2.0
mkdir -v build
cd       build
../libstdc++-v3/configure           \
    --host=$LFS_TGT                 \
    --build=$(../config.guess)      \
    --prefix=/usr                   \
    --disable-multilib              \
    --disable-nls                   \
    --disable-libstdcxx-pch         \
    --with-gxx-include-dir=/tools/$LFS_TGT/include/c++/11.2.0
make
make DESTDIR=$LFS install
cd ../..
rm -rf gcc-11.2.0
```

## 6. Cross Compiling Temporary Tools

## 7. Entering Chroot and Building Additional Temporary Tools
```
su -
chown -R root:root $LFS/{usr,lib,var,etc,bin,sbin,tools}
case $(uname -m) in
  x86_64) chown -R root:root $LFS/lib64 ;;
esac
mkdir -pv $LFS/{dev,proc,sys,run}
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
mount -v --bind /dev $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run
if [ -h $LFS/dev/shm ]; then
  mkdir -pv $LFS/$(readlink $LFS/dev/shm)
fi
```
* enter chroot environment
```
chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    /bin/bash --login
mkdir -pv /{boot,home,mnt,opt,srv}
mkdir -pv /etc/{opt,sysconfig}
mkdir -pv /lib/firmware
mkdir -pv /media/{floppy,cdrom}
mkdir -pv /usr/{,local/}{include,src}
mkdir -pv /usr/local/{bin,lib,sbin}
mkdir -pv /usr/{,local/}share/{color,dict,doc,info,locale,man}
mkdir -pv /usr/{,local/}share/{misc,terminfo,zoneinfo}
mkdir -pv /usr/{,local/}share/man/man{1..8}
mkdir -pv /var/{cache,local,log,mail,opt,spool}
mkdir -pv /var/lib/{color,misc,locate}

ln -sfv /run /var/run
ln -sfv /run/lock /var/lock

install -dv -m 0750 /root
install -dv -m 1777 /tmp /var/tmp

ln -sv /proc/self/mounts /etc/mtab
cat > /etc/hosts << EOF
127.0.0.1  localhost $(hostname)
::1        localhost
EOF

cat > /etc/passwd << "EOF"
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/dev/null:/usr/bin/false
daemon:x:6:6:Daemon User:/dev/null:/usr/bin/false
messagebus:x:18:18:D-Bus Message Daemon User:/run/dbus:/usr/bin/false
uuidd:x:80:80:UUID Generation Daemon User:/dev/null:/usr/bin/false
nobody:x:99:99:Unprivileged User:/dev/null:/usr/bin/false
EOF

cat > /etc/group << "EOF"
root:x:0:
bin:x:1:daemon
sys:x:2:
kmem:x:3:
tape:x:4:
tty:x:5:
daemon:x:6:
floppy:x:7:
disk:x:8:
lp:x:9:
dialout:x:10:
audio:x:11:
video:x:12:
utmp:x:13:
usb:x:14:
cdrom:x:15:
adm:x:16:
messagebus:x:18:
input:x:24:
mail:x:34:
kvm:x:61:
uuidd:x:80:
wheel:x:97:
nogroup:x:99:
users:x:999:
EOF

echo "tester:x:101:101::/home/tester:/bin/bash" >> /etc/passwd
echo "tester:x:101:" >> /etc/group
install -o tester -d /home/tester

exec /usr/bin/bash --login

touch /var/log/{btmp,lastlog,faillog,wtmp}
chgrp -v utmp /var/log/lastlog
chmod -v 664  /var/log/lastlog
chmod -v 600  /var/log/btmp
```
* install temporary tools

* clean up
```
rm -rf /usr/share/{info,man,doc}/*
find /usr/{lib,libexec} -name \*.la -delete
rm -rf /tools
exit
```
* backup (IF NEEDED)
```
umount $LFS/dev/pts
umount $LFS/{sys,proc,run,dev}
cd $LFS
tar -cJpf $HOME/lfs-temp-tools-11.1.tar.xz .
```
* restore (NOT NEEDED IF EVERYTHING IS IN PLACE ALREADY)
```
cd $LFS
rm -rf ./*
tar -xpf $HOME/lfs-temp-tools-11.1.tar.xz
```
* enter chroot environment as described above and before that mount everything that is needed
```
mknod -m 600 $LFS/dev/console c 5 1
mknod -m 666 $LFS/dev/null c 1 3
mount -v --bind /dev $LFS/dev
mount -v --bind /dev/pts $LFS/dev/pts
mount -vt proc proc $LFS/proc
mount -vt sysfs sysfs $LFS/sys
mount -vt tmpfs tmpfs $LFS/run

chroot "$LFS" /usr/bin/env -i   \
    HOME=/root                  \
    TERM="$TERM"                \
    PS1='(lfs chroot) \u:\w\$ ' \
    PATH=/usr/bin:/usr/sbin     \
    /bin/bash --login
```
* ...........

## Perform actions needed for 42 task
* hostname
```
echo "gtristan" > /etc/hostname
```
* set locale
```
cat > /etc/profile << "EOF"
# Begin /etc/profile

export LANG=ru_RU.utf8_RU.UTF-8

# End /etc/profile
EOF
```
* console settings
```
cat > /etc/sysconfig/console << "EOF"
# Begin /etc/sysconfig/console

UNICODE="1"
KEYMAP="ruwin_alt-UTF-8"
FONT="UniCyrExt_8x16"

# End /etc/sysconfig/console
```
* clock settings
```
cat > /etc/sysconfig/clock << "EOF"
# Begin /etc/sysconfig/clock

UTC=3

# Set this to any options you might need to give to hwclock,
# such as machine hardware clock type for Alphas.
CLOCKPARAMS=

# End /etc/sysconfig/clock
EOF
```
* inittab
```
cat > /etc/inittab << "EOF"
# Begin /etc/inittab

id:3:initdefault:

si::sysinit:/etc/rc.d/init.d/rc S

l0:0:wait:/etc/rc.d/init.d/rc 0
l1:S1:wait:/etc/rc.d/init.d/rc 1
l2:2:wait:/etc/rc.d/init.d/rc 2
l3:3:wait:/etc/rc.d/init.d/rc 3
l4:4:wait:/etc/rc.d/init.d/rc 4
l5:5:wait:/etc/rc.d/init.d/rc 5
l6:6:wait:/etc/rc.d/init.d/rc 6

ca:12345:ctrlaltdel:/sbin/shutdown -t1 -a -r now

su:S016:once:/sbin/sulogin

1:2345:respawn:/sbin/agetty --noclear tty1 9600
2:2345:respawn:/sbin/agetty tty2 9600
3:2345:respawn:/sbin/agetty tty3 9600
4:2345:respawn:/sbin/agetty tty4 9600
5:2345:respawn:/sbin/agetty tty5 9600
6:2345:respawn:/sbin/agetty tty6 9600

# End /etc/inittab
EOF
```
* inputrc
```
cat > /etc/inputrc << "EOF"
# Begin /etc/inputrc
# Modified by Chris Lynn <roryo@roryo.dynup.net>

# Allow the command prompt to wrap to the next line
set horizontal-scroll-mode Off

# Enable 8bit input
set meta-flag On
set input-meta On

# Turns off 8th bit stripping
set convert-meta Off

# Keep the 8th bit for display
set output-meta On

# none, visible or audible
set bell-style none

# All of the following map the escape sequence of the value
# contained in the 1st argument to the readline specific functions
"\eOd": backward-word
"\eOc": forward-word

# for linux console
"\e[1~": beginning-of-line
"\e[4~": end-of-line
"\e[5~": beginning-of-history
"\e[6~": end-of-history
"\e[3~": delete-char
"\e[2~": quoted-insert

# for xterm
"\eOH": beginning-of-line
"\eOF": end-of-line

# for Konsole
"\e[H": beginning-of-line
"\e[F": end-of-line

# End /etc/inputrc
EOF
```
* /etc/shells
```
cat > /etc/shells << "EOF"
# Begin /etc/shells

/bin/sh
/bin/bash

# End /etc/shells
EOF
```
## Making LFS system bootable
* fstab
```
cat > /etc/fstab << "EOF"
# Begin /etc/fstab

# file system  mount-point  type     options             dump  fsck
#                                                              order

/dev/sda1      /boot        devtmpfs defaults            0     2
/dev/sda2      /            ext4     defaults            1     1
/dev/sda3      none         swap     pri=1               0     0
proc           /proc        proc     nosuid,noexec,nodev 0     0
sysfs          /sys         sysfs    nosuid,noexec,nodev 0     0
devpts         /dev/pts     devpts   gid=5,mode=620      0     0
tmpfs          /run         tmpfs    defaults            0     0
devtmpfs       /dev         devtmpfs mode=0755,nosuid    0     0

# End /etc/fstab
EOF
```
* change distribution name  - insert in Makefile 
```
EXTRAVERSION = -gtristan
```
* mount boot partition
```
mount /dev/sda1 /boot
```
* change ownership of linux directory
```
chown -R 0:0 ../linux-5.16.9
```
* modprobe
```
install -v -m755 -d /etc/modprobe.d
cat > /etc/modprobe.d/usb.conf << "EOF"
# Begin /etc/modprobe.d/usb.conf

install ohci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i ohci_hcd ; true
install uhci_hcd /sbin/modprobe ehci_hcd ; /sbin/modprobe -i uhci_hcd ; true

# End /etc/modprobe.d/usb.conf
EOF
```
* install GRUB
```
grub-install --target i386-pc /dev/sda
```
* creating GRUB configuration file
```
cat > /boot/grub/grub.cfg << "EOF"
# Begin /boot/grub/grub.cfg
set default=0
set timeout=5

insmod ext2
set root=(hd0,1)

menuentry "GNU/Linux, Linux 5.16.9-gtristan" {
        linux   /vmlinuz-5.16.9-gtristan root=/dev/sda2 ro
}
EOF
```
* release info
```
cat > /etc/lsb-release << "EOF"
DISTRIB_ID="Linux From Scratch"
DISTRIB_RELEASE="11.1"
DISTRIB_CODENAME="gtristan"
DISTRIB_DESCRIPTION="Linux From Scratch"
EOF

cat > /etc/os-release << "EOF"
NAME="Linux From Scratch"
VERSION="11.1"
ID=gtristan
PRETTY_NAME="Linux From Scratch 11.1"
VERSION_CODENAME="gtristan"
EOF
```
* the end
```
logout

umount -v $LFS/dev/pts
umount -v $LFS/dev
umount -v $LFS/run
umount -v $LFS/proc
umount -v $LFS/sys

umount -v $LFS/boot
umount -v $LFS
```

* add disk to virtualbox
```
sudo vboxmanage internalcommands createrawvmdk -filename ~/LFS.vmdk -rawdisk /dev/sda
```

* configure network
1. Create NatNetwork in preferences of the Virtualbox with standard settings
2. Make port forwarding there:
```
Host Port - 5522
Guest IP 192.168.56.2
Guest Port - 22
```
3. Create in tools of VirtualBox a host-only adapter with standard settings:
```
Configure Adapter Manually
IPv4 Address - 192.168.56.1
IPv4 Network Mask - 255.255.255.0
```
4. In VM settings create two adapters
```
Adapter 1 - NAT Network / NatNetwork
Adapter 2 - Host-only adapter / vboxnet0
```
5. Inside the VM itself create 2 interface files:
```
# cat /etc/sysconfig/ifconfig.enp0s3
ONBOOT=yes
IFACE=enp0s3
SERVICE=ipv4-static
IP=10.0.2.2
GATEWAY=10.0.2.1
PREFIX=24
BROADCAST=10.0.2.255

# cat /etc/sysconfig/ifconfig.enp0s8
ONBOOT=yes
IFACE=enp0s8
SERVICE=ipv4-static
IP=192.168.56.2
GATEWAY=192.168.56.1
PREFIX=24
BROADCAST=192.168.56.255
```
