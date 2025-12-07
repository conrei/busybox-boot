A simple guide for building a bootable busybox iso image with network access.

This reference follows arm64 fedora as the host system.<br>
Adjust as needed for your distribution and architecture.

## Dependencies
```bash
sudo dnf install gcc make git flex bison glibc-static \
    openssl-devel ncurses-devel elfutils-libelf-devel \
    xorriso mtools dosfstools cpio grub2-tools-extra \
    bc wget grub2-efi-aa64-modules grub2-efi-aa64-cdboot
      # or grub2-efi-x64-modules grub2-efi-x64-cdboot
```

Set up a working directory:
```bash
mkdir ~/busybox-iso
cd ~/busybox-iso
export WORK_DIR=$(pwd)
```

## Building the kernel
Fetch the kernel source tarball:
```bash
wget https://cdn.kernel.org/pub/linux/kernel/v6.x/linux-6.18.tar.xz
tar -xf linux-6.18.tar.xz
cd linux-6.18
```

Configure and build the kernel:
```bash
make defconfig
make kvm_guest.config

# just in case
./scripts/config --enable CONFIG_BLK_DEV_INITRD
./scripts/config --enable CONFIG_RD_GZIP

make -j$(nproc) Image.gz
```
The build will take some time as we're using the default configuration. It's highly recommended to slim the kernel down with a custom configuration to reduce its size and build time, but that's out of the scope of this guide. A good starting point would be reading about [modprobed-db](https://wiki.archlinux.org/title/Modprobed-db).

Copy the kernel image:
```bash
# change arch accordingly
cp arch/arm64/boot/Image.gz $WORK_DIR/kernel.gz
cd $WORK_DIR
```

## Building busybox
Fetch the busybox source tarball:
```bash
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2
tar -xf busybox-1.37.0.tar.bz2
cd busybox-1.37.0

export BUSYBOX_DIR=$(pwd)
```
If the official busybox mirror is down, try [this mirror](https://github.com/conrei/busybox-mirror/releases) instead.

Configure and build busybox:
```bash
make defconfig

# force static build
sed -i 's/^# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config
# disable traffic control
sed -i 's/^CONFIG_TC=y/CONFIG_TC=n/' .config

make -j$(nproc) install
```

If the build fails due to `libbb/hash_md5_sha.c`, you're most likely on arm64 and have to disable sha hardware acceleration:
```bash
sed -i 's/^CONFIG_SHA1_HWACCEL=y/# CONFIG_SHA1_HWACCEL is not set/' .config
sed -i 's/^CONFIG_SHA256_HWACCEL=y/# CONFIG_SHA256_HWACCEL is not set/' .config

# rebuild
make -j$(nproc) install
```

## Setting up the filesystem
Create the root of the filesystem:
```bash
cd $WORK_DIR
mkdir rootfs
cd rootfs
```

Copy busybox and create some necessary directories:
```bash
cp -a $BUSYBOX_DIR/_install/* .
mkdir dev proc sys etc
```

Set up networking with busybox's udhcpc:
```bash
mkdir -p usr/share/udhcpc

cp $BUSYBOX_DIR/examples/udhcp/simple.script usr/share/udhcpc/default.script
chmod +x usr/share/udhcpc/default.script

cat > etc/hosts <<EOF
127.0.0.1   localhost
::1         localhost
EOF

touch etc/resolv.conf
```

Set up busybox as our init system:
```bash
ln -s bin/busybox init
mkdir etc/init.d

cat > etc/init.d/rcS <<EOF
#!/bin/sh
mount -t proc proc /proc
mount -t sysfs sysfs /sys
mount -t devtmpfs devtmpfs /dev

dmesg -n 1

ip link set lo up
ip link set eth0 up

udhcpc -b -i eth0 -s /usr/share/udhcpc/default.script
EOF

chmod +x etc/init.d/rcS

cat > etc/inittab <<EOF
::sysinit:/etc/init.d/rcS
::respawn:/bin/cttyhack /bin/sh
::restart:/init
::ctrlaltdel:/sbin/reboot
EOF
```

Pack everything into a cpio archive:
```bash
find . -print0 \
| cpio --null -ov --format=newc -R +0:+0 \
| gzip -9 > $WORK_DIR/initramfs.cpio.gz
```

## Setting up GRUB
This section is irrelevant if you choose to use a different bootloader.

Set up the boot directory:
```bash
cd $WORK_DIR
mkdir -p iso/boot/grub
```

Copy kernel and ramfs:
```
cp kernel.gz iso/boot/
cp initramfs.cpio.gz iso/boot/
```

Configure GRUB and build the iso image:
```bash
cat > iso/boot/grub/grub.cfg <<EOF
set default=0
set timeout=0

menuentry "busybox" {
    linux /boot/kernel.gz console=tty0 console=ttyAMA0 earlycon
    initrd /boot/initramfs.cpio.gz
}
EOF

grub2-mkrescue -o busybox.iso iso/
```
If you're building for x86_64, `ttyAMA0` isn't necessary.

## And that's it
Test if it boots correctly:
```bash
qemu-system-aarch64 \
    -M virt -cpu host --enable-kvm \
    -m 512M \
    -cdrom busybox.iso
    # -bios /usr/share/edk2/aarch64/QEMU_EFI.fd
```
