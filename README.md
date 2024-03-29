# CoreOS ARM64 Notes

2016.02.22

## Info

The releases here are unofficial ARM64 CoreOS disk images that I've build for testing CoreOS on QEMU, the 96boards HiKey developer board, and the Huawei D02 development board.  For info on the HiKey board see https://www.96boards.org/products/ce/hikey/.  For info on the D02 board see http://open-estuary.org/.

You can always find the latest version of this document, and some other useful
technical documents at https://github.com/glevand/hikey-coreos/

Please send comments to <geoff@infradead.org>.  I can also be reached on
freenode/#coreos as geoff-

## License

Permission is granted to copy, distribute and/or modify this document under the
terms of the GNU Free Documentation License, Version 1.3 published by the Free
Software Foundation; with no Invariant Sections, no Front-Cover Texts, and no
Back-Cover Texts. A copy of the license is included in the section entitled "GNU
Free Documentation License".

## QEMU

The CoreOS SDK includes qemu-system-aarch64.  This should be enough for general development work, and is what I recommend.  Newer distros may have a packaged qemu-system-aarch64 that you can use.

To run CoreOS with QEMU you'll need UEFI firmware built for qemu-system-aarch64.  I use the binary releases provided by Linaro:

    wget http://releases.linaro.org/components/kernel/uefi-linaro/latest/release/qemu64/QEMU_EFI.fd

I run QEMU as shown below.  This should work with either a pre-built arm64 coreos_developer_image or one build with the CoreOS SDK using 'build_image board=arm64-usr dev':

    qemu-system-aarch64 -machine virt -cpu cortex-a57 -machine type=virt -nographic -m 2048 \
      -bios QEMU_EFI.fd \
      -drive if=none,id=blk,file=arm64_coreos_developer_image_${version}.bin \
      -device virtio-blk-device,drive=blk \
      -net user,hostfwd=tcp::10022-:22 \
      -device virtio-net-device,vlan=0 \
      -fsdev local,id=host_dev,path=${path_to}/coreos-sdk,security_model=none \
      -device virtio-9p-device,fsdev=host_dev,mount_tag=host_dev

The -fsdev option will mount a host directory tree in the QEMU ARM64 guest, which allows you to work with files from both the host and target.

In the QEMU guest use something like this to mount the host directory:

    mount -t 9p -o trans=virtio host_dev ${path_to}/coreos-sdk

Or better:

    echo "host_dev ${path_to}/coreos-sdk 9p trans=virtio 0 0" >> /etc/fstab

## Hikey Board

NOTE: The Hikey board is not supported for these releases:

    935.0.0+2016-01-27-1049-a1
    949.0.0+2016-02-09-0944-a1
    962.0.0+2016-02-19-0947-a1

### SD Card Setup

Need 2.0 GiB or larger Micro SD card for HiKey.

    cat arm64_coreos_developer_image_${version}.bin.xz | xz -d > arm64_coreos_developer_image_${version}.bin
    dd if=arm64_coreos_developer_image_${version}.bin of=/dev/sdX bs=4M oflag=sync

### eMMC Setup

If not installed, install UEFI firmware, Grub bootloader, and a Debian system image to the HiKey eMMC.  The Debian install on eMMC will be for setup and recovery.  See https://github.com/96boards/documentation/wiki/HiKeyUEFI and https://github.com/96boards/documentation/wiki/LatestSnapshots.

Download the needed files from:

    http://builds.96boards.org

You can try my program-hikey utility included in this release if you like.  You'll need to set the builds_root variable.  This command should install everything needed to boot to Debian from the eMMC:

    program-hikey --firmware --boot --system

### Grub menu

Boot the Hikey without an SD card installed or choose the grub 'eMMC' boot option.  Add a CoreOS menu item like the following to the eMMC grub config file:

    debian # vi /boot/grub/grub.cfg

    menuentry 'CoreOS: SD Card' {
    set root='hd1,1'
    linux /coreos/vmlinuz-a console=tty0 console=ttyAMA3,115200 earlycon=pl011,0xf7113000 root=/dev/mmcblk1p9 rootwait ro efi=noruntime
    devicetree /coreos/hi6220-hikey.dtb
    }

I recommend you increase the grub.cfg 'timeout' value, and maybe set the 'default' to your 'CoreOS: SD Card' entry.

You can edit the eMMC grub config from CoreOS with something like:

    coreos # mkdir /tmp/grub
    coreos # mount /dev/mmcblk0p6 /tmp/grub/
    coreos # vi /tmp/grub/grub/grub.cfg

### Start Up

Insert SD card into HiKey, power on, choose 'CoreOS: SD Card' menu item.

## General Notes

### CoreOS Login

  user='core', password='c'

### Debugging

To inspect the CoreOS disk image use somthing like:

    sudo kpartx -v -a -s arm64_coreos_developer_image_${version}.bin
    sudo mount /dev/mapper/loopXp1 /tmp/boot
    sudo mount /dev/mapper/loopXp3 /tmp/usr
    sudo mount /dev/mapper/loopXp9 /tmp/rootfs

### Building

See The README at https://github.com/glevand/coreos--manifest for info on building CoreOS with my development repositories.
