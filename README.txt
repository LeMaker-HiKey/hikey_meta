            Hikey Snappy 16 Guide


Prepare enviroment
------------------

install ubuntu xenial 16.04 and install below packages:

$ apt-get install snapcraft parted kpartx \
  dosfstools squashfs-tools android-tools-fsutils \
  gcc-aarch64-linux-gnu


Get source of hikey snaps
-------------------------

 - Export PATH to build snappy image

$ export SNAPPY_PATH=/path/snap

 - Get hikey meta:

$ cd $SNAPPY_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_snappy_meta.git

 - Get hikey kernel snap:

$ cd $SNAPPY_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_linux.git

 - Get hikey gadget snap:

$ cd $SNAPPY_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_gadget_snap.git


Build and install u-boot to hikey
---------------------------------

Because snappy only supports u-boot on arm platform, but hikey now uses UEFI as its default 
bootloader, we have to switch hikey to u-boot.

 - Get mainline u-boot source:

$ cd $SNAPPY_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_u-boot.git u-boot
$ cd u-boot
$ git checkout -b hikey_snappy.test origin/hikey_snappy

 - Get other source code:

$ cd $SNAPPY_PATH
$ git clone https://github.com/96boards/edk2.git
$ git clone https://github.com/96boards/arm-trusted-firmware.git
$ git clone https://github.com/96boards/burn-boot.git
$ git clone git://github.com/96boards/l-loader.git

And follow u-boot/board/hisilicon/hikey/README to compile and burn to hikey board.

Note: before generate l-load.bin. please modify generate_ptable.sh, replace "system" with
"writable" in the following line.

 sgdisk -n -E -t 9:8300 -u 9:FC56E345-2E8E-49AE-B2F8-5B9D263FE377 -c 9:"system" ${TEMP_FILE}

After all are done successfully, we have a hikey board running u-boot.


Build hikey kernel snap
-----------------------

 - Build the kernel snap:

$ cd $SNAPPY_PATH
$ cd hikey_linux
$ git checkout -b snappy_v4.4 origin/snappy_v4.4
$ snapcraft --target-arch arm64 snap
$ ls hikey-kernel_4.4.0_arm64.snap
$ cp hikey-kernel_4.4.0_arm64.snap ../


Build hikey gadget snap
-----------------------

 - Build the gadget snap:

$ cd $SNAPPY_PATH
$ rm -rf hikey_gadget_snap/.git
$ snapcraft snap hikey_gadget_snap

If we want to change uboot.env, please use mkenvimage to re-generate uboot.env
after modify hikey_gadget_snap/uboot.env.in:

$ ./u-boot/tools/mkenvimage -s 0x20000 -o hikey_gadget_snap/uboot.env \
  -r hikey_gadget_snap/uboot.env.in


Build snappy image
------------------

$ cd $SNAPPY_PATH
$ sudo ./hikey_meta_github/ubuntu-device-flash core 16 --developer-mode --channel edge \
  --os ubuntu-core --kernel ./hikey-kernel_4.4.0_arm64.snap \
  --gadget ./hikey-snappy-gadget_0.1_arm64.snap -o hikey-snappy.img


Burn snappy image to SD card
----------------------------

Insert a sd card to PC linux and burn snappy image to SD card:

$ sudo dd if=hikey-snappy.img of=/dev/sdx bs=32M

Please replace /dev/sdx with your SD card device file.


Burn snappy rootfs to EMMC FLASH
--------------------------------

 - Generate writable image from snappy image:

$ cd $SNAPPY_PATH
$ mkdir writable
$ sudo kpartx -av hikey-snappy.img
$ sudo mount /dev/mapper/loop0p2 writable
$ sudo make_ext4fs -L writable -l 1500M -s writable.img writable/
$ sudo umount writable
$ sudo kpartx -d hikey-snappy.img 

Note:
 1. loop0p2 is the loop device of the second partition of hikey-snappy.img, it should
 be replaced with your loop device of kpartx mapping.


 - Burn writable image to EMMC FLASH:

$ cd $SNAPPY_PATH
$ sudo ./burn-boot/hisi-idt.py -d /dev/ttyUSB1 --img1=l-loader/l-loader.bin
$ sudo fastboot flash ptable l-loader/ptable-linux-8g.img
$ sudo fastboot flash writable writable.img
