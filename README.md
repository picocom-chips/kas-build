# kas-build
## Build Images:
PC805 build system is based on kas-container. Kas-container is a script that allows you to run kas inside a Docker container without installing any dependencies on your host system. To use kas-container, you need to have Docker installed on your system and have access to the internet. 
### Clone the Build System
```
git clone https://github.com/picocom-chips/kas-build.git
```
### Build the PC805 Images
```
cd kas-build
./kas-container build pc805-poky.yml
```
Because the build system needs to clone plenty of repositories over the internet, the build may take several hours.
### Images Created by the Build
If the build command finished without error, you can get the final image at the following folder `build/tmp/deploy/images/pc805`. And the following images will be created:
- Secondary Program Loader: u-boot-spl.bin
- RISC-V Supervisor Binary Interface: fw_jump.bin
- Bootloader: u-boot.bin
- Linux Kernel: fitImage-pc805.bin
- Kernel Modules: modules-pc805.tgz
- Rootfs: picocom-image-pc805.ext4
## Flash Images
### Setup TFTP Server on Windows:
#### Download Tftpd64
```
https://bitbucket.org/phjounin/tftpd64/downloads/Tftpd64-4.64-setup.exe
```
This TFTP server is very simple, you just need to run it by double click, and then configure the file root in the GUI.
#### Copy Images to the TFTP Server Root Directory
Download the following files from build server output directory to tftp server root directory.
```
From Build Server: build/tmp/deploy/images/pc805
To TFTP Server: C:\temp (according to your configuration)
```
### Bring Up the PC805 Board:
If the PC805 chip has FSBL in the OTP. The FSBL will try to load u-boot-spl image from UART firstly. 
#### Load u-boot-spl image to chipset with UART
When you see "C" on the UART console, send the u-boot-spl.bin with YMODEM protocol in your UART terminal software. (TeraTerm is one of the free software which supports YMODEM).
#### Load u-boot image to chipset with UART
U-boot-SPL still doesn't have the ability to write SPI-NAND flash, so we need load u-boot.bin to chipset through UART. SPL image use PC805 GPIO0 as boot selection temporarily, when it detected the GPIO0 is pull down, it will print "C" on UART console and wait for host side send the u-boot.bin image through UART by YMODEM protocol.
### Flash Images to SPI-NAND in U-Boot:
```
dhcp
setenv serverip 172.16.18.68 // change the server ip according to your configuration

tftp u-boot-spl.bin
nand erase 0 $(filesize)
nand write 90100000 0 $(filesize)

tftp fw_jump.bin
nand erase 80000 $(filesize)
nand write 90100000 80000 $(filesize)

tftp u-boot.bin
nand erase 100000 $(filesize)
nand write 90100000 100000 $(filesize)

tftp fitImage-pc805.bin
nand erase 200000 $(filesize)
nand write 90100000 200000 $(filesize)

# UBIFS rootfs
tftp picocom-image-pc805.ubi
nand erase 800000 1f800000
nand write 90100000 800000 $(filesize)

# EXT4 rootfs
tftp picocom-image-pc805.ext4
nand erase 800000 1f800000
nand write 90100000 800000 $(filesize)
```
### Flash Images to SPI-NAND in Kernel Command Line: (for on-line upgrade only)
We also can flash the images to spi-nand flash under linux kernel command line:
```
tftp -g -r u-boot-spl.bin 172.16.16.170
flashcp -v u-boot-spl.bin /dev/mtd0

tftp -g -r fw_jump.bin 172.16.16.170
flashcp -v fw_jump.bin /dev/mtd1

tftp -g -r u-boot.bin 172.16.16.170
flashcp -v u-boot.bin /dev/mtd2

tftp -g -r fitimage-pc805.bin 172.16.16.170
flashcp -v fitimage-pc805.bin /dev/mtd4

# UBIFS rootfs
flash_erase /dev/mtd5 0 0
tftp -g -r picocom-image-pc805.ubi 172.16.16.170
flashcp -v picocom-image-pc805.ubi /dev/mtd5

# EXT4 rootfs
tftp -g -r picocom-image-pc805.ext4 172.16.16.170
flashcp -v picocom-image-pc805.ext4 /dev/mtd5
```
## Run Images:
Note: If above steps completed succesfully, no need flash images again after reboot board. 
### Expand Rootfs Size to Whole MTD Partition
*Notes: This steps only need for EXT4 rootfs*

If you finally managed to get into spi-nand rootfs, you will find the rootfs size is very small, but actually, our nand flash size is nearly 500M bytes. This is because in order to make the flash image smaller, we limited the image size to 32M. So we need to expand the rootfs to whole mtd partition in normal use cases.
```
resize2fs /dev/mtdblock5
```
### Modify and Save U-Boot Envs
At the beginning stage of the boot, you can enter the U-Boot shell by pressing keys to stop the normal boot procedure. 
#### Set IP Address in U-Boot and Save
```
setenv ipaddr 172.16.16.171
saveenv
```
#### Set MAC address in U-Boot and Save
```
setenv ethaddr ca:ff:ee:12:34:56
saveenv
```
#### Set TFTP Server IP in U-Boot and Save
```
setenv serverip 172.16.18.68
saveenv
```
#### Set Bootargs for UBIFS/EXT4 rootfs
```
# UBIFS
setenv bootargs console=ttyS0,115200 earlycon=sbi loglevel=8 rootfstype=ubifs ubi.mtd=rootfs root=ubi0:pc805-rootfs
saveenv

# EXT4
setenv bootargs console=ttyS0,115200 earlycon=sbi loglevel=8 root=/dev/mtdblock5
saveenv
```
## Build Driver and Application in Docker Shell
```
cd kas-build
./kas-container shell pc805-poky.yml
```
### Cross Compiler Path
```
/build/tmp/sysroots-components/x86_64/gcc-cross-riscv32/usr/bin/riscv32-poky-linux/riscv32-poky-linux-gcc
```
### Linux Header Path
```
/build/tmp/work-shared/pc805/kernel-source
```
## SDK of the PC805 Board
If you want to use the riscv32 cross compiler out of the docker environment, you can build a SDK of PC805 board and use it in any Linux Distributions.
### Build the SDK
```
./kas-container shell pc805-poky.yml -c "bitbake picocom-image -c populate_sdk"
```
Once the build is finished without error, you can get the installable SDK from `build/tmp/deploy/sdk/poky-glibc-x86_64-picocom-image-riscv32-pc805-toolchain-4.0.13.sh`
### Install the SDK
```
./poky-glibc-x86_64-picocom-image-riscv32-pc805-toolchain-4.0.13.sh
```
### Use the SDK
Run the following command, the cross compiler will be add to the system PATH, and you can use it directly from shell.
```
. /opt/poky/4.0.13/environment-setup-riscv32-poky-linux
```

### Use the SDK to compile the kernel module
*Notes: The following command only need run once after you install the SDK*
```
cd -P /opt/poky/4.0.13/sysroots/riscv32-poky-linux/usr/src/kernel
. /opt/poky/4.0.13/environment-setup-riscv32-poky-linux
make scripts
make prepare
```
Then cd to your kernel module folder to do do the compile. The kernel build directory is located at `/opt/poky/4.0.13/sysroots/riscv32-poky-linux/usr/src/kernel`. Don't forget to make the relevant change in your kernel module Makefile.
```
cd <kernel modoule folder>
. /opt/poky/4.0.13/environment-setup-riscv32-poky-linux
make
```
### Check the GCC version
```
$ riscv32-poky-linux-gcc --version
riscv32-poky-linux-gcc (GCC) 11.4.0
Copyright (C) 2021 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```
## Notes:
1. Give some patience when you see the following output at first kernel boot, Because generating ssh key takes some time.
```
Starting OpenBSD Secure Shell server: sshd
  generating ssh RSA host key...
```
