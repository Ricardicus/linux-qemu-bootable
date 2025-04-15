# Linux under QEMU

This document will describe how to build and run
your own distribution of Linux using QEMU.

# Prerequisits

I will assume you use Debian package manager and are on a Linux system.
This does not mean this cannot be done any other way, it just happens
to be the setup I had when going throught this process and I only document
what I know.

Start by installing some prerequisites:

```bash
# QEMU for x86
sudo apt install -y qemu-system-x86
# Building Linux
sudo apt install -y build-essential libncurses-dev bison flex libssl-dev libelf-dev bc
```

I might be missing something, maybe some additional apt installing will be needed on your part.

## Introduction

We will need to do three things:

- Build Linux bootable image
- Create an initial ramdisk
- Create a disk for persistant storage

Choose a unique place in your file system. Lets call this location "the base directory".

```bash
mkdir base-directory
cd base-directory
# Lets get to work
```

## Building Linux from source

Start fresh on a unique location in your filesystem.
Here, clone Linux (i've chosen to work with the tag v6.15-rc2
but you can choose any other release):

```bash
# Clone the Linux source tree, this might take some time
git clone --branch v6.15-rc2 https://github.com/torvalds/linux.git
cd linux
```

Linux is a complex program and therefore has a complex build.
There are multiple ways you can customize your Linux OS.
A file called ".config" determines the behaviour of your system.
But what should that .config look like?

For the everyday, assume the default is good enought for me, usecase,
you can create a default .config like this:

```bash
make defconfig
```

If you have done this many times before, and want to tweak the
Linux OS to meet your demands you can customize it like this:

```bash
# This is if you need to customize your build
make menuconfig
```

Now the .config file probably exist, and you can build the OS.
This can take a while. Build like this:

```bash
make -j$(nproc)
```

You have now built the kernel! Congratulations!

The bootable kernel image is found here (in case x86_64 system):

```bash
arch/x86_64/boot/bzImage
```

This is your file! We will give this file to QEMU, it is the bootable 
program. Now we will need the file system. 

## Initial ramdisk

The filesystem can live on disk, of a physical harddrive and it can
also live in RAM. For Linux to boot up, there needs to be a filesystem.
We will now create an initial ramdisk that is a filesystem that will
be loaded into memory.

We will work with "alpine" because it has a package manager and that
is nice.

Stand in the base directory and do this:

```bash
wget https://dl-cdn.alpinelinux.org/alpine/v3.21/releases/x86_64/alpine-minirootfs-3.21.3-x86_64.tar.gz
mkdir alpine-rootfs
cd alpine-rootfs
tar -xzf ../alpine-minirootfs-3.21.3-x86_64.tar.gz 
```

Your system should "start" from somewhere after boot. We need a "init" script, create one
here called "init" and copy-paste this content in it:

```bash
#!/bin/sh
echo "Initializing my system"

# Setting up internet
sleep 1
ip link set eth0 up
udhcpc -i eth0

# random number generator
mknod -m 666 /dev/urandom c 1 9 

# Mounting some devices
mkdir -p /proc /sys /dev /tmp
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
chmod 1777 /tmp

export PATH=$PATH:/usr/libexec/gcc/x86_64-alpine-linux-musl/14.2.0

# Disk mounting
mkdir -p /mnt
mount /dev/sda /mnt

# Copying some of the states into RAM
# - uncomment after having created this disk
#   and populated with programs from apk

#cp -a /mnt/usr/* /usr/
#cp -a /mnt/etc/* /etc/
#cp -a /mnt/lib/* /lib/

# Setting up SSH
#chown root:root /var/empty
#chmod 755 /var/empty
#cp -r /mnt/.ssh /.ssh
#ssh-keygen -A
#/usr/sbin/sshd

# Launch /bin/sh
exec /bin/sh
```

This init script does some basic things. We will uncomment some parts of it 
in the future. We will get back to this init-script.
Lets change make a change in the file system so that the owner of these files
are the root:

```bash
chmod a+x init
sudo chown -R root:root .
```

Now lets create a initial ramdisk!
This can be done like this:

```bash
sudo find . -print0 | sudo cpio --null -ov --format=newc | gzip -9 > ../initramfs-alpine.cpio.gz 
```

We will store the init ramdisk in the base directory.

# Booting

Lets boot!

We will make sure to install some programs also, because in Alpine we have access to "apk".
The init ramdisk only resides in RAM, which means that none of the memory is persistent.
Lets create a persistent memory drive, a storage disk.

```bash
# Create a 2 GB large filesystem
qemu-img create -f qcow2 alpine-root.qcow2 2G
```

## Sanity check

Just to check that everything is fine thus far, the base directory shoud now look like this:

```bash
$ ls -l
total 7064
-rw-r--r--  1 rickard rickard 3507952 Feb 14 00:04 alpine-minirootfs-3.21.3-x86_64.tar.gz
drwxr-xr-x 19 root    root       4096 Apr 15 13:32 alpine-rootfs
-rw-r--r--  1 rickard rickard  196640 Apr 15 13:39 alpine-root.qcow2
-rw-r--r--  1 rickard rickard 3503954 Apr 15 13:36 initramfs-alpine.cpio.gz
drwxr-xr-x 27 rickard rickard    4096 Apr 15 13:18 linux
```

(yes, my name is rickard...)

## Lets boot for the first time, populate the disk then reboot

Create in the base directory this script, alpine-run.sh:

```bash
#!/bin/bash

# Define disk image path
DISK_IMAGE="alpine-root.qcow2"

# Check if disk image exists
if [ ! -f "$DISK_IMAGE" ]; then
  echo "Disk image not found. Creating $DISK_IMAGE..."
  qemu-img create -f qcow2 "$DISK_IMAGE" 1G
fi

# Launch QEMU with the disk
qemu-system-x86_64 \
  -m 2048 \
  -kernel linux/arch/x86_64/boot/bzImage \
  -initrd initramfs-alpine.cpio.gz \
  -nographic \
  -append "console=ttyS0 rdinit=/init" \
  -net nic -net user,hostfwd=tcp::2222-:22 \
  -drive file="$DISK_IMAGE",format=qcow2,index=0,media=disk
```

Make it executable:

```bash
chmod a+x alpine-run.sh
```

Lets try to boot:

```bash
./alpine-run.sh
```

Did it work? Try some commands.. for example "ls":

```bash
...
[    2.311770] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x23fa7089fe7, max_idle_ns: 440795281784 ns
[    2.314312] clocksource: Switched to clocksource tsc
Initializing my system
[    2.514567] input: ImExPS/2 Generic Explorer Mouse as /devices/platform/i8042/serio1/input/input3
[    3.367206] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
udhcpc: started, v1.37.0
[    3.433160] ip (57) used greatest stack depth: 13536 bytes left
udhcpc: broadcasting discover
udhcpc: broadcasting select for 10.0.2.15, server 10.0.2.2
udhcpc: lease of 10.0.2.15 obtained from 10.0.2.2, lease time 86400
/bin/sh: can't access tty; job control turned off
~ # ls -al
total 8
drwxr-xr-x   19 root     root           420 Apr 15 11:58 .
drwxr-xr-x   19 root     root           420 Apr 15 11:58 ..
-rw-------    1 root     root             7 Apr 15 11:58 .ash_history
drwxr-xr-x    2 root     root          1680 Feb 13 23:04 bin
drwxr-xr-x    7 root     root          2300 Apr 15 11:58 dev
drwxr-xr-x   17 root     root           760 Apr 15 11:58 etc
drwxr-xr-x    2 root     root            40 Feb 13 23:04 home
-rwxr-xr-x    1 root     root           778 Apr 15 11:55 init
drwxr-xr-x    6 root     root           160 Feb 13 23:04 lib
drwxr-xr-x    5 root     root           100 Feb 13 23:04 media
drwxr-xr-x    2 root     root            40 Feb 13 23:04 mnt
drwxr-xr-x    2 root     root            40 Feb 13 23:04 opt
dr-xr-xr-x  107 root     root             0 Apr 15 11:58 proc
drwx------    2 root     root            40 Feb 13 23:04 root
drwxr-xr-x    3 root     root            60 Feb 13 23:04 run
drwxr-xr-x    2 root     root          1260 Feb 13 23:04 sbin
drwxr-xr-x    2 root     root            40 Feb 13 23:04 srv
dr-xr-xr-x   12 root     root             0 Apr 15 11:58 sys
drwxrwxrwt    2 root     root            40 Feb 13 23:04 tmp
drwxr-xr-x    7 root     root           140 Feb 13 23:04 usr
drwxr-xr-x   11 root     root           260 Feb 13 23:04 var
```

# Populate the hard disk

While in the shell session, lets mount the drive:

```bash
apk update
apk add e2fsprogs
mkfs.ext4 /dev/sda
mkdir -p /mnt
mount /dev/sda /mnt
```

Lets install some programs we want and then store them into
the mounted drive:

```bash
apk add openssl openssh
cp -r /usr /mnt/usr
cp -r /lib /mnt/lib
cp -r /etc /mnt/etc
mkdir /mnt/.ssh
echo "export PS1='\u@\h:\w\$ '" > /mnt/.profile
```

Edit the /mnt/etc/ssh/sshd_config so that it is like the content of
the "sshd_config"-file provided next to this README.md file.
In general, set:

```text
PermitRootLogin prohibit-password
PubkeyAuthentication yes
AuthorizedKeysFile      /.ssh/authorized_keys
PasswordAuthentication no
Match User root
    ForceCommand /bin/sh -i
```

Also, place a public key (I will assume you know how to create such keys)
under the authorized_keys file. Say the key is id_rsa.pub:

```bash
echo "ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAACAQDVXCTnJGUd2NQ7nTbD66Wqgyh2lOUIjyWii8yg8YFZJmbF78itewJneLE7rtQPu0jvZi8Lkl4HG8ArXLsObBxMZ0gOqm54OVkuBrMnKdyrMWUz2iKHm6iCl4Q9UKR89dG7vQdnAYbLGb7imxh3IbrRFVLDpoRe2Xc+y9eHjeunuL2piOa4e/NSojZM1N1ciRxURARjdxePH6jxYW/AyeEn8HjjYcaIX91CVuoZyoIw2bOgE3+8zjf3EsMNYGtimcE410uT+hEdOpuvAVf/PW88yZrSxVF5uhoCfa4aGMNucC7QyxZESvgpxSW6DVeYdb7rRfXSUFRb5aaJVTv0tcPubjnc1fKIwUgjWnIC/ay+3OU0Q9eVkv4I1fERbjLG4oiDvqVhGxwfb7/HKmnPI2Jmor+5jJ27yXkpNFqhd0z/bv2NZdR8D+FG9WEL03awgQbIlVqgnP0MzUrq1Z9AJwVHlELyLWk8F7YWeokvBPgzZj9f7N2WdIfA/QMghlxfZbqQXdud0tmf9pTxfm0ibjhUnffmPoZuqL2iywtry+G9XYp9bBgC9GY1W/1yoP0hJ9i0jb73ZlLmvM4we73n6xXWwucZ41UxpHa7hQPiUcJ3UGco1nGrqjeMrCi/Q74jGiHM8vYcH4AtSZ7ipWFOqxXEn/b4n9doIRtWaHJamJi+cw== rickard.hallerback@gmail.com" > /mnt/.ssh/authorized_keys
```

Openssh includes the ssh-daemon "sshd". This program is not
included from the beginning, so we can copy this program
into RAM from disk on start. Now, revist the init-script.
Uncomment the lines where we copy over stuff, and do ssh-relate
things. Now make the init-script like this:

```bash
#!/bin/sh
echo "Initializing my system"

# Setting up internet
sleep 1
ip link set eth0 up
udhcpc -i eth0

# random number generator
mknod -m 666 /dev/urandom c 1 9 

# Mounting some devices
mkdir -p /proc /sys /dev /tmp
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs none /dev
chmod 1777 /tmp

export PATH=$PATH:/usr/libexec/gcc/x86_64-alpine-linux-musl/14.2.0

# Disk mounting - uncomment after having created this disk
mkdir -p /mnt
mount /dev/sda /mnt

# Copying some of the states into RAM
cp -a /mnt/usr/* /usr/
cp -a /mnt/etc/* /etc/
cp -a /mnt/lib/* /lib/

# Setting up SSH
chown root:root /var/empty
chmod 755 /var/empty
cp -r /mnt/.ssh /root/.ssh
ssh-keygen -A
/usr/sbin/sshd

# Launch /bin/sh
exec /bin/sh
```

Rebuild the init ramdisk:

```bash
cd alpine-rootfs
sudo find . -print0 | sudo cpio --null -ov --format=newc | gzip -9 > ../initramfs-alpine.cpio.gz 
```

Now reboot again:

```bash
./alpine-run.sh
```

In another shell, try to ssh into the device now like this:

```bash
ssh -p 2222 -i ~/.ssh/id_rsa_example root@localhost
```

If you use id_rsa_example, make sure that the content of id_rsa_example.pub is in /mnt/.ssh/authorized_keys .

This should work now!
Hope this was informative.
