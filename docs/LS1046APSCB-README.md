# LS1046APSCB Yocto Build Environment (KAS)

### Build Rootfs
```
kas build kas/nxp/ls1046apscb-dev.yml
```
### Build Command for ls1046apscb
```
kas shell kas/nxp/ls1046apscb-dev.yml -c "bitbake secure-boot-qoriq"
kas shell kas/nxp/ls1046apscb-dev.yml -c "bitbake qoriq-composite-firmware"
kas shell kas/nxp/ls1046apscb-dev.yml -c "bitbake generate-boottgz"
kas build kas/nxp/ls1046apscb-dev.yml
```
### Build Command for ls1046apscbc
```
kas shell kas/nxp/ls1046apscbc-dev.yml -c "bitbake secure-boot-qoriq"
kas shell kas/nxp/ls1046apscbc-dev.yml -c "bitbake qoriq-composite-firmware"
kas shell kas/nxp/ls1046apscbc-dev.yml -c "bitbake generate-boottgz"
kas build kas/nxp/ls1046apscbc-dev.yml
```
### Build Command for ls1046apxcp
```
kas shell kas/nxp/ls1046apxcp-dev.yml -c "bitbake secure-boot-qoriq"
kas shell kas/nxp/ls1046apxcp-dev.yml -c "bitbake qoriq-composite-firmware"
kas shell kas/nxp/ls1046apxcp-dev.yml -c "bitbake generate-boottgz"
kas build kas/nxp/ls1046apxcp-dev.yml
```
### Format SD card
```
./flex-installer -i pf -d /dev/sdc
```
### Install Images to SD Card
```
sudo ./flex-installer -f firmware_ls1046apscb_uboot_sdboot.img -b boot_ls1046apscb_lts_6.6.tgz -r pico-sdk-ls1046a-ls1046apscb.rootfs.tar.gz -d /dev/sdc
```
### Set Ethernet MAC Address in U-Boot
```
setenv ethaddr 9F:86:A2:50:6F:BF
```
### Boot FIT image from TFTP server
```
tftp 80000000 kernel-fsl-ls1046a-pscbc-sdk.itb
bootm 80000000#ls1046apscbc
```
### Set MAC Address in U-boot
```
setenv eth10addr 00:E0:0C:00:02:0a
setenv eth11addr 00:E0:0C:00:02:0b
setenv eth12addr 00:E0:0C:00:02:0c
setenv eth13addr 00:E0:0C:00:02:0d
setenv eth14addr 00:E0:0C:00:02:0e
setenv eth15addr 00:E0:0C:00:02:0f
setenv ethaddr 00:E0:0C:00:02:00
setenv eth1addr 00:E0:0C:00:02:01
setenv eth2addr 00:E0:0C:00:02:02
setenv eth3addr 00:E0:0C:00:02:03
setenv eth4addr 00:E0:0C:00:02:04
setenv eth5addr 00:E0:0C:00:02:05
setenv eth6addr 00:E0:0C:00:02:06
setenv eth7addr 00:E0:0C:00:02:07
setenv eth8addr 00:E0:0C:00:02:08
setenv eth9addr 00:E0:0C:00:02:09
```
### Enable Ethernet Device in /etc/network/interfaces
```
# for ls1046apacb
auto fm1-mac5
iface fm1-mac5 inet dhcp

# for ls1046apscbc
auto fm1-mac6
iface fm1-mac6 inet dhcp
```
### Patch for meta-qoriq layer
- patches/ls1046apscb-meta-qoriq.patch

### Install packages through dev netwrok
#### Update /etc/apt/sources.list file as the following
```
deb [trusted=yes] http://101.69.253.219:8000/ls1046apscbc/deb/all ./
deb [trusted=yes] http://101.69.253.219:8000/ls1046apscbc/deb/aarch64 ./
deb [trusted=yes] http://101.69.253.219:8000/ls1046apscbc/deb/aarch64-qoriq ./
deb [trusted=yes] http://101.69.253.219:8000/ls1046apscbc/deb/ls1046apscbc ./
```
#### Set the correct system time if haven't done so
```
date -s "2025-07-02 15:52:00"
```
#### Install the packages
```
apt update
apt install gdb
```