# kas-build
## PC805 Boot Flow
PC805 is a Linux-capable RISC-V SOC. Like a typical embedded system boot flow, the following component was involved to support boot system into Linux land.
- BootROM: hardcoded in the chip
- FSBL: first stage boot loader, flashed in the OTP zone of the chip
- SPL: secondory program loader, flashed in the SPI-NAND flash
- U-Boot: binary of U-Boot, a wide used boot loader, flashed in the SPI-NAND flash
- SBI: RISC-V Supervisor Binary Interface, OpenSBI is one of the typical implementing, flashed in the SPI-NAND flash 
- Kernel: Linux kernel image, flashed in the SPI-NAND flash
Here is a simple diagram show steps of system boot into Linux land.

![image](https://github.com/picocom-chips/kas-build/assets/149779491/5896b29d-159b-4323-a14b-b2b95f083a6f)

## Build Images:
PC805 build system is based on kas and Yocto. Kas-container is a script that allows you to run kas and yocto inside a Docker container without installing any dependencies on your host system. To use kas-container, you need to have Docker installed on your system and have access to the internet. 
You can get a guide about how to install Docker at [Install Docker Engine](https://docs.docker.com/engine/install/).
With this all-in-one build environment, you can create all the binaries which is necessary to boot a PC805 chip into Linux land.
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
If the build command finished without error, you can get the final image at the following folder `build/tmp/deploy/images/pc805`. The following images will be created:
- Secondary Program Loader: u-boot-spl.bin
- RISC-V Supervisor Binary Interface: fw_jump.bin
- Bootloader: u-boot.bin
- Linux Kernel: fitImage-pc805.bin
- Kernel Modules: modules-pc805.tgz
- Rootfs(UBIFS): picocom-image-pc805.ubi
- Rootfs(EXT4FS): picocom-image-pc805.ext4
## Flash Images
### Setup TFTP Server on Windows:
#### Download Tftpd64
```
https://bitbucket.org/phjounin/tftpd64/downloads/Tftpd64-4.64-setup.exe
```
This TFTP server is very simple, you just need to run it by double click, and then configure the file root in the GUI.
![image](https://github.com/picocom-chips/kas-build/assets/149779491/9e952d32-d792-4689-8faa-a9cf54da0193)
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
U-boot-SPL still doesn't have the ability to write SPI-NAND flash, so we need load u-boot.bin to chipset through UART. SPL image use PC805 GPIO-31 as boot selection temporarily, when it detected the GPIO-31 is pull down, it will print "C" on UART console and wait for host side send the u-boot.bin image through UART by YMODEM protocol.
### Flash Images to SPI-NAND in U-Boot (For UBIFS Rootfs)
```
dhcp
setenv serverip <your-tftp-server-ip>

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
```
### Flash Images to SPI-NAND in Kernel Console (For UBIFS Rootfs)
We also can flash the images to spi-nand flash under linux kernel console:
```
tftp -g -r u-boot-spl.bin <your-tftp-server-ip>
flashcp -v u-boot-spl.bin /dev/mtd0

tftp -g -r fw_jump.bin <your-tftp-server-ip>
flashcp -v fw_jump.bin /dev/mtd1

tftp -g -r u-boot.bin <your-tftp-server-ip>
flashcp -v u-boot.bin /dev/mtd2

tftp -g -r fitimage-pc805.bin <your-tftp-server-ip>
flashcp -v fitimage-pc805.bin /dev/mtd4

# UBIFS rootfs
flash_erase /dev/mtd5 0 0
tftp -g -r picocom-image-pc805.ubi <your-tftp-server-ip>
flashcp -v picocom-image-pc805.ubi /dev/mtd5
```
> [!CAUTION]
> The following flash image steps only need when you want to use EXT4 type of rootfs. Since UBIFS was designed for unmanaged flash memory devices, it was confirmed has better performance over EXT4. We strong recommand you chose UBIFS rootfs.
> ### Flash Images to SPI-NAND in U-Boot (For EXT4 Rootfs)
> ```
> dhcp
> setenv serverip <your-tftp-server-ip>
> 
> tftp u-boot-spl.bin
> nand erase 0 $(filesize)
> nand write 90100000 0 $(filesize)
> 
> tftp fw_jump.bin
> nand erase 80000 $(filesize)
> nand write 90100000 80000 $(filesize)
> 
> tftp u-boot.bin
> nand erase 100000 $(filesize)
> nand write 90100000 100000 $(filesize)
> 
> tftp fitImage-pc805.bin
> nand erase 200000 $(filesize)
> nand write 90100000 200000 $(filesize)
> 
> # EXT4 rootfs
> tftp picocom-image-pc805.ext4
> nand erase 800000 1f800000
> nand write 90100000 800000 $(filesize)
> ```
> ### Flash Images to SPI-NAND in Kernel Console (For EXT4 Rootfs)
> We also can flash the images to spi-nand flash under linux kernel console:
> ```
> tftp -g -r u-boot-spl.bin <your-tftp-server-ip>
> flashcp -v u-boot-spl.bin /dev/mtd0
> 
> tftp -g -r fw_jump.bin <your-tftp-server-ip>
> flashcp -v fw_jump.bin /dev/mtd1
> 
> tftp -g -r u-boot.bin <your-tftp-server-ip>
> flashcp -v u-boot.bin /dev/mtd2
> 
> tftp -g -r fitimage-pc805.bin <your-tftp-server-ip>
> flashcp -v fitimage-pc805.bin /dev/mtd4
> 
> # EXT4 rootfs
> tftp -g -r picocom-image-pc805.ext4 <your-tftp-server-ip>
> flashcp -v picocom-image-pc805.ext4 /dev/mtd5
> ```
> ### Expand Rootfs Size to Whole MTD Partition
> *Notes: This steps only need for EXT4 rootfs*
>
> Once you finally managed to run into Linux console with EXT4 rootfs, you will find the rootfs size is very small. However, The nand flash on the EVB is nearly 500M bytes. This is because we limited the EXT4 rootfs image size to 64M, in order to make it easier to download and flash. So we need to expand the rootfs to take whole mtd partition for normal use cases.
> ```
> resize2fs /dev/mtdblock5
> ```
## Run Images:
*Note: If above steps completed succesfully, no need flash images again after reboot board.*

When you power on the EVB board with images flashed, you will be able to see the following printing on DEBUG UART.
```log
CCCC
FSBL b7b84d3945ed (Mon Nov 13 10:23:01 CST 2023)
dl-110
MID 0xc8 (0x45 0x0 0x0)
Model GD5F4GQ6REYIG
Spinand GigaDevice
k1n
clrjp>

U-Boot SPL 2020.10 (Nov 22 2023 - 07:53:56 +0000)
Boot select detect: register val = 0xffe07fff
Trying to boot from NAND
Loading U-Boot from 0x00100000 (size 0x000c0000) to 0x90000000
call nand_deselect

U-Boot 2020.10 (Nov 20 2023 - 07:03:49 +0000)

DRAM:  512 MiB
NAND:  512 MiB
Loading Environment from NAND... OK
In:    uart@8212000
Out:   uart@8212000
Err:   uart@8212000
Net:   eth0: ethernet@8480000
Hit any key to stop autoboot:  0
Reading 524288 byte(s) (256 page(s)) at offset 0x00000000
Reading 6291456 byte(s) (3072 page(s)) at offset 0x00000000
## Loading kernel from FIT Image at 91000000 ...
   Using 'conf-picocom_pc805.dtb' configuration
   Trying 'kernel-1' kernel subimage
     Description:  Linux kernel
     Type:         Kernel Image
     Compression:  gzip compressed
     Data Start:   0x91000110
     Data Size:    3052951 Bytes = 2.9 MiB
     Architecture: RISC-V
     OS:           Linux
     Load Address: 0x80400000
     Entry Point:  0x80400000
     Hash algo:    sha256
     Hash value:   5f32150f0718bfe7e024496d4f89a0c0b5953040e07447d82dce615647ddc5db
   Verifying Hash Integrity ... sha256+ OK
## Loading fdt from FIT Image at 91000000 ...
   Using 'conf-picocom_pc805.dtb' configuration
   Trying 'fdt-picocom_pc805.dtb' fdt subimage
     Description:  Flattened Device Tree blob
     Type:         Flat Device Tree
     Compression:  uncompressed
     Data Start:   0x912e97bc
     Data Size:    8596 Bytes = 8.4 KiB
     Architecture: RISC-V
     Load Address: 0x82200000
     Hash algo:    sha256
     Hash value:   a8e5415afdb695b781573ae02fd9d6854fdc37f58127b915d72c93a558d625a3
   Verifying Hash Integrity ... sha256+ OK
   Loading fdt from 0x912e97bc to 0x82200000
   Booting using the fdt blob at 0x82200000
   Uncompressing Kernel Image
   Using Device Tree in place at 82200000, end 82205193

Starting kernel ...


OpenSBI v1.1-57-g26fdcca
   ____                    _____ ____ _____
  / __ \                  / ____|  _ \_   _|
 | |  | |_ __   ___ _ __ | (___ | |_) || |
 | |  | | '_ \ / _ \ '_ \ \___ \|  _ < | |
 | |__| | |_) |  __/ | | |____) | |_) || |_
  \____/| .__/ \___|_| |_|_____/|____/_____|
        | |
        |_|

Platform Name             : Picocom PC805
Platform Features         : medeleg
Platform HART Count       : 1
Platform IPI Device       : ---
Platform Timer Device     : pc805_plmt @ 0Hz
Platform Console Device   : uart8250
Platform HSM Device       : ---
Platform PMU Device       : ---
Platform Reboot Device    : ---
Platform Shutdown Device  : ---
Firmware Base             : 0x80000000
Firmware Size             : 152 KB
Runtime SBI Version       : 1.0

Domain0 Name              : root
Domain0 Boot HART         : 0
Domain0 HARTs             : 0*
Domain0 Region00          : 0x80000000-0x8003ffff ()
Domain0 Region01          : 0x00000000-0xffffffff (R,W,X)
Domain0 Next Address      : 0x80400000
Domain0 Next Arg1         : 0x82200000
Domain0 Next Mode         : S-mode
Domain0 SysReset          : yes

Boot HART ID              : 0
Boot HART Domain          : root
Boot HART Priv Version    : v1.11
Boot HART Base ISA        : rv32imafdcx
Boot HART ISA Extensions  : none
Boot HART PMP Count       : 16
Boot HART PMP Granularity : 8
Boot HART PMP Address Bits: 30
Boot HART MHPM Count      : 4
Boot HART MIDELEG         : 0x00000222
Boot HART MEDELEG         : 0x0000b109
[    0.000000] OF: fdt: Ignoring memory range 0x80000000 - 0x80400000
[    0.000000] Linux version 5.4.147-pc805 (oe-user@oe-host) (gcc version 11.4.0 (GCC)) #1 Sat Jan 6 12:20:43 UTC 2024
[    0.000000] earlycon: sbi0 at I/O port 0x0 (options '')
[    0.000000] printk: bootconsole [sbi0] enabled
[    0.000000] initrd not found or empty - disabling initrd
[    0.000000] Reserved memory: created CMA memory pool at 0x000000009e000000, size 32 MiB
[    0.000000] OF: reserved mem: initialized node linux,cma, compatible id shared-dma-pool
[    0.000000] Zone ranges:
[    0.000000]   Normal   [mem 0x0000000080400000-0x000000009fffffff]
[    0.000000] Movable zone start for each node
[    0.000000] Early memory node ranges
[    0.000000]   node   0: [mem 0x0000000080400000-0x000000008fffffff]
[    0.000000]   node   0: [mem 0x000000009a000000-0x000000009fffffff]
[    0.000000] Initmem setup node 0 [mem 0x0000000080400000-0x000000009fffffff]
[    0.000000] On node 0 totalpages: 89088
[    0.000000]   Normal zone: 1016 pages used for memmap
[    0.000000]   Normal zone: 0 pages reserved
[    0.000000]   Normal zone: 89088 pages, LIFO batch:15
[    0.000000] SBI specification v1.0 detected
[    0.000000] SBI implementation ID=0x1 Version=0x10001
[    0.000000] SBI v0.2 TIME extension detected
[    0.000000] SBI v0.2 IPI extension detected
[    0.000000] SBI v0.2 RFENCE extension detected
[    0.000000] pcpu-alloc: s0 r0 d32768 u32768 alloc=1*32768
[    0.000000] pcpu-alloc: [0] 0
[    0.000000] Built 1 zonelists, mobility grouping on.  Total pages: 88072
[    0.000000] Kernel command line: console=ttyS0,115200 earlycon=sbi loglevel=8 rootfstype=ubifs ubi.mtd=rootfs root=ubi0:pc805-rootfs
[    0.000000] Dentry cache hash table entries: 65536 (order: 6, 262144 bytes, linear)
[    0.000000] Inode-cache hash table entries: 32768 (order: 5, 131072 bytes, linear)
[    0.000000] Sorting __ex_table...
[    0.000000] mem auto-init: stack:off, heap alloc:off, heap free:off
[    0.000000] Memory: 313692K/356352K available (3743K kernel code, 204K rwdata, 1044K rodata, 140K init, 227K bss, 9892K reserved, 32768K cma-reserved)
[    0.000000] NR_IRQS: 72, nr_irqs: 72, preallocated irqs: 0
[    0.000000] plic: mapped 146 interrupts with 1 handlers for 2 contexts.
[    0.000000] riscv_timer_init_dt: Registering clocksource cpuid [0] hartid [0]
[    0.000000] clocksource: riscv_clocksource: mask: 0xffffffffffffffff max_cycles: 0xe2b8193d7, max_idle_ns: 881590406657 ns
[    0.000012] sched_clock: 64 bits at 30MHz, resolution 32ns, wraps every 4398046511090ns
[    0.008524] Console: colour dummy device 80x25
[    0.012982] Calibrating delay loop (skipped), value calculated using timer frequency.. 61.44 BogoMIPS (lpj=30720)
[    0.023169] pid_max: default: 32768 minimum: 301
[    0.028114] Mount-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.035389] Mountpoint-cache hash table entries: 1024 (order: 0, 4096 bytes, linear)
[    0.048635] devtmpfs: initialized
[    0.055796] random: get_random_u32 called from bucket_table_alloc.isra.0+0x124/0x13e with crng_init=0
[    0.111035] DMA-API: preallocated 65586 debug entries
[    0.116032] DMA-API: debugging enabled by kernel config
[    0.121290] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
[    0.131044] futex hash table entries: 256 (order: -1, 3072 bytes, linear)
[    0.141569] NET: Registered protocol family 16
[    0.171451] pps_core: LinuxPPS API ver. 1 registered
[    0.176373] pps_core: Software ver. 5.3.6 - Copyright 2005-2007 Rodolfo Giometti <giometti@linux.it>
[    0.185556] PTP clock support registered
[    0.190796] clocksource: Switched to clocksource riscv_clocksource
[    0.216149] NET: Registered protocol family 2
[    0.221257] IP idents hash table entries: 8192 (order: 4, 65536 bytes, linear)
[    0.230112] tcp_listen_portaddr_hash hash table entries: 512 (order: 0, 4096 bytes, linear)
[    0.238463] TCP established hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.246305] TCP bind hash table entries: 4096 (order: 2, 16384 bytes, linear)
[    0.253512] TCP: Hash tables configured (established 4096 bind 4096)
[    0.260229] UDP hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.266762] UDP-Lite hash table entries: 256 (order: 0, 4096 bytes, linear)
[    0.274209] NET: Registered protocol family 1
[    0.280724] workingset: timestamp_bits=30 max_order=17 bucket_order=0
[    0.347512] dw_axi_dmac_platform 10c00000.dmac: DesignWare AXI DMA Controller, 8 channels
[    0.358556] Serial: 8250/16550 driver, 3 ports, IRQ sharing disabled
[    0.367041] printk: console [ttyS0] disabled
[    0.371340] 8212000.uart: ttyS0 at MMIO 0x8212000 (irq = 70, base_baud = 20400000) is a 16550A
[    0.379945] printk: console [ttyS0] enabled
[    0.379945] printk: console [ttyS0] enabled
[    0.388318] printk: bootconsole [sbi0] disabled
[    0.388318] printk: bootconsole [sbi0] disabled
[    0.399658] dw_spi_mmio 8204000.spi: Synopsys DWC APB SSI v4.02
[    0.405691] dw_spi_mmio 8204000.spi: Detected FIFO size: 22 bytes
[    0.411837] dw_spi_mmio 8204000.spi: Detected 32-bits max data frame size
[    0.419212] dw_spi_mmio 8204000.spi: registered master spi0
[    0.425191] spi spi0.0: setup mode 0, 8 bits/w, 10000000 Hz max --> 0
[    0.432546] dw_spi_mmio 8204000.spi: registered child spi0.0
[    0.438687] dw_spi_mmio 8206000.spi: Synopsys DWC APB SSI v4.02
[    0.444706] dw_spi_mmio 8206000.spi: Detected FIFO size: 22 bytes
[    0.450851] dw_spi_mmio 8206000.spi: Detected 32-bits max data frame size
[    0.457752] dw_spi_mmio 8206000.spi: DMA init failed
[    0.463144] dw_spi_mmio 8206000.spi: registered master spi1
[    0.469136] spi spi1.0: setup mode 0, 8 bits/w, 10000000 Hz max --> 0
[    0.476461] dw_spi_mmio 8206000.spi: registered child spi1.0
[    0.482588] dw_spi_mmio 820c000.spi: Synopsys DWC APB SSI v4.02
[    0.488610] dw_spi_mmio 820c000.spi: Detected FIFO size: 50 bytes
[    0.494755] dw_spi_mmio 820c000.spi: Detected 32-bits max data frame size
[    0.502122] dw_spi_mmio 820c000.spi: registered master spi2
[    0.508144] spi spi2.0: setup mode 0, 8 bits/w, 40000000 Hz max --> 0
[    0.515200] spi-nand spi2.0: GigaDevice SPI NAND was found.
[    0.520863] spi-nand spi2.0: 512 MiB, block size: 128 KiB, page size: 2048, OOB size: 128
[    0.530154] 6 fixed-partitions partitions found on MTD device spi2.0
[    0.536610] Creating 6 MTD partitions on "spi2.0":
[    0.541468] 0x000000000000-0x000000080000 : "spl"
[    0.549311] 0x000000080000-0x000000100000 : "opensbi"
[    0.557460] 0x000000100000-0x0000001c0000 : "uboot"
[    0.565629] 0x0000001c0000-0x000000200000 : "ubootenv"
[    0.573642] 0x000000200000-0x000000800000 : "kernel"
[    0.585734] 0x000000800000-0x000020000000 : "rootfs"
[    0.960198] dw_spi_mmio 820c000.spi: registered child spi2.0
[    0.966642] libphy: Fixed MDIO Bus: probed
[    0.971515] dwc-eth-dwmac 8480000.ethernet: PTP uses main clock
[    0.977534] dwc-eth-dwmac 8480000.ethernet: no reset control found
[    0.984345] dwc-eth-dwmac 8480000.ethernet: User ID: 0x11, Synopsys ID: 0x52
[    0.991494] dwc-eth-dwmac 8480000.ethernet:  DWMAC4/5
[    0.996604] dwc-eth-dwmac 8480000.ethernet: DMA HW capability register supported
[    1.004053] dwc-eth-dwmac 8480000.ethernet: RX Checksum Offload Engine supported
[    1.011500] dwc-eth-dwmac 8480000.ethernet: Wake-Up On Lan supported
[    1.017907] dwc-eth-dwmac 8480000.ethernet: Enable RX Mitigation via HW Watchdog Timer
[    1.025877] dwc-eth-dwmac 8480000.ethernet: device MAC address 82:40:d7:3a:01:53
[    1.033588] libphy: stmmac: probed
[    1.041954] NET: Registered protocol family 17
[    1.046523] NET: Registered protocol family 15
[    1.051158] Bridge firewalling registered
[    1.055691] l2tp_core: L2TP core driver, V2.0
[    1.060139] 8021q: 802.1Q VLAN Support v1.8
[    1.069663] DCCP: Activated CCID 2 (TCP-like)
[    1.074110] DCCP: Activated CCID 3 (TCP-Friendly Rate Control)
[    1.080248] sctp: Hash tables configured (bind 1024/1024)
[    1.104686] ubi0: attaching mtd5
[    5.820854] ubi0: scanning is finished
[    5.854659] ubi0: attached mtd5 (name "rootfs", size 504 MiB)
[    5.860498] ubi0: PEB size: 131072 bytes (128 KiB), LEB size: 126976 bytes
[    5.867422] ubi0: min./max. I/O unit sizes: 2048/2048, sub-page size 2048
[    5.874254] ubi0: VID header offset: 2048 (aligned 2048), data offset: 4096
[    5.881259] ubi0: good PEBs: 4032, bad PEBs: 0, corrupted PEBs: 0
[    5.887397] ubi0: user volume: 1, internal volumes: 1, max. volumes count: 128
[    5.894664] ubi0: max/mean erase counter: 2/1, WL threshold: 4096, image sequence number: 1638215424
[    5.903839] ubi0: available PEBs: 0, total reserved PEBs: 4032, PEBs reserved for bad PEB handling: 80
[    5.913775] ubi0: background thread "ubi_bgt0d" started, PID 54
[    5.920091] ttyS0 - failed to request DMA
[    5.926352] UBIFS (ubi0:0): Mounting in unauthenticated mode
[    6.007927] UBIFS (ubi0:0): recovery needed
[    6.109893] UBIFS (ubi0:0): recovery deferred
[    6.114525] UBIFS (ubi0:0): UBIFS: mounted UBI device 0, volume 0, name "pc805-rootfs", R/O mode
[    6.123370] UBIFS (ubi0:0): LEB size: 126976 bytes (124 KiB), min./max. I/O unit sizes: 2048 bytes/2048 bytes
[    6.133330] UBIFS (ubi0:0): FS size: 499904512 bytes (476 MiB, 3937 LEBs), journal size 9023488 bytes (8 MiB, 72 LEBs)
[    6.144064] UBIFS (ubi0:0): reserved for root: 0 bytes (0 KiB)
[    6.149948] UBIFS (ubi0:0): media format: w4/r0 (latest is w5/r0), UUID F7659AD7-2D16-459C-A561-A7CCD922D61F, small LPT model
[    6.165196] VFS: Mounted root (ubifs filesystem) readonly on device 0:14.
[    6.175912] devtmpfs: mounted
[    6.180103] Freeing unused kernel memory: 140K
[    6.184615] This architecture does not have kernel memory protection.
[    6.191097] Run /sbin/init as init process
INIT: version 3.01 booting
Starting udev
[    7.875052] udevd[87]: starting version 3.2.10
[    7.905737] random: udevd: uninitialized urandom read (16 bytes read)
[    7.930890] random: udevd: uninitialized urandom read (16 bytes read)
[    7.937618] random: udevd: uninitialized urandom read (16 bytes read)
[    8.064328] udevd[88]: starting eudev-3.2.10
[    9.080547] UBIFS (ubi0:0): completing deferred recovery
[    9.223502] UBIFS (ubi0:0): background thread "ubifs_bgt0_0" started, PID 112
[    9.234304] UBIFS (ubi0:0): deferred recovery completed
hwclock: can't open '/dev/misc/rtc': No such file or directory
Fri Mar  9 12:34:56 UTC 2018
hwclock: can't open '/dev/misc/rtc': No such file or directory
[   10.582417] urandom_read: 1 callbacks suppressed
[   10.582433] random: dd: uninitialized urandom read (512 bytes read)
INIT: Entering runlevel: 5
Configuring network interfaces... [   11.145854] dwc-eth-dwmac 8480000.ethernet eth0: PHY [stmmac-0:01] driver [SMSC LAN8710/LAN8720]
[   11.154909] cma: cma_alloc(cma (ptrval), count 2, align 1)
[   11.161603] cma: cma_alloc(): returned (ptrval)
[   11.166574] cma: cma_alloc(cma (ptrval), count 2, align 1)
[   11.172262] cma: cma_alloc(): returned (ptrval)
[   11.192457] dwmac4: Master AXI performs any burst length
[   11.197859] dwc-eth-dwmac 8480000.ethernet eth0: No Safety Features support found
[   11.205409] dwc-eth-dwmac 8480000.ethernet eth0: PTP not supported by HW
[   11.212176] dwc-eth-dwmac 8480000.ethernet eth0: configuring for phy/rmii link mode
[   11.222179] 8021q: adding VLAN 0 to HW filter on device eth0
udhcpc: started, v1.35.0
udhcpc: broadcasting discover
[   13.314892] dwc-eth-dwmac 8480000.ethernet eth0: Link is Up - 100Mbps/Full - flow control rx/tx
udhcpc: broadcasting discover
udhcpc: broadcasting select for 10.13.52.27, server 10.70.148.32
udhcpc: lease of 10.13.52.27 obtained from 10.70.148.32, lease time 691200
/etc/udhcpc.d/50default: Adding DNS 10.24.148.30
/etc/udhcpc.d/50default: Adding DNS 10.20.148.30
done.
[   16.793887] random: crng init done
Starting OpenBSD Secure Shell server: sshd
done.
hwclock: can't open '/dev/misc/rtc': No such file or directory
Starting syslogd/klogd: done

Poky (Yocto Project Reference Distro) 4.0.13 pc805 /dev/ttyS0

pc805 login:
```
> [!NOTE]
> Give some patience when you see the following output at first kernel boot, Because generating ssh key takes some time.
> ```
> Starting OpenBSD Secure Shell server: sshd
>   generating ssh RSA host key...
> ```
### Username and Password of Linux Console
You can login to Linux console use username **root** without password.

### Remote SSH Access
Once the system running into Linux system, it can get the IP address from DHCP server. The SSH server is enabled in the system by default. You also can access to the board through SSH protocol.
```
ssh root@<your-board-ip>
```

### Things We Can Do in U-Boot Console
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
#### Set Bootargs for UBIFS/EXT4 Rootfs
```
# UBIFS
setenv bootargs console=ttyS0,115200 earlycon=sbi loglevel=8 rootfstype=ubifs ubi.mtd=rootfs root=ubi0:pc805-rootfs
saveenv

# EXT4
setenv bootargs console=ttyS0,115200 earlycon=sbi loglevel=8 root=/dev/mtdblock5
saveenv
```
## Build Driver and Application in Docker Shell
### Enter the Console of Kas Build System
```bash
cd kas-build
./kas-container shell pc805-poky.yml
# build Linux kernel
bitbake virtual/kernel
# build u-boot
bitbake u-boot-picocom
```
### Build Single Component Without Enter Kas Shell
You also can build a single component without enter kas shell through oneline combined command. For example, you can build Linux kenrel with:
```
./kas-container shell pc805-poky.yml -c "bitbake virtual/kernel"
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
### Use the Cross Toolchain Compile C/C++ Program
When you use the cross toolchain to compile any C/C++ program, don't forget to specify the **--sysroot** parameter to let the cross compiler search headers and libraries in right path.
```
riscv32-poky-linux-gcc --sysroot=/home/attina/workspace/pico/poky/4.0.13/sysroots/riscv32-poky-linux hello-world.c -o hello-world
```
