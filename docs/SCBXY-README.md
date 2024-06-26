# SCBXY Yocto Build Environment
The build system of scbxy is mainly based on Yocto and Kas build system. The bitbake layer of RK3399 is from rockchip chip open source project.
## Build
### Get the source code
```bash
git clone https://github.com/picocom-chips/kas-build.git
```
### Build the scbxy image
```bash
cd kas-build
./kas-container build kas/rockchip/scbxy-rel.yml
```
### Images created by build system
The final image file located at `build/latest` folder after run the build.
```bash
update.img
```
### Get the yocto layers without build
```
./kas-container checkout kas/rockchip/scbxy-rel.yml
```
## Flash
Currently, scbxy can boot from SD card. You can craete a bootable SD card by tools `SDDiskTool` which was released in RK3399 SDK package.
![img_v3_028h_575234b7-f43c-4d0f-a4b5-ed122a8e302g](https://github.com/picocom-chips/kas-build/assets/149779491/3fd1e1f3-b4c6-4151-8bba-f1eb7268960a)


## Customization
### Enable PREEMPT RT kernel
We are maintaining two kernel branches for RK3399.
- picocom/rk3399-master: for normal kernel scheduler
- picocom/rk3399-rt-master: for realtime preempt scheduler

You can get the kernel source code from [Github](https://github.com/picocom-chips/linux). Local copy of Linux kernel can be found at `build/tmp/work-shared/scbxy/kernel-source` after you run the build command.

You can modify the board config file of scbxy to switch bwtween the RT and NON-RT kernel. The board config file is located at `layers/meta-rockchip/conf/machine/scbxy.conf`. The `layers/meta-rockchip` folder will be cloned from Picocom github repository automatically when you run build or checkout layer command.

#### Variable to enable/disable preempt rt kernel
```
# layers/meta-rockchip/conf/machine/scbxy.conf
ENABLE_PREEMPT_RT = 'true'
```
### Build SDK for The SCBXY board
If you want to use the aarch64 cross compiler out of the docker environment, you can build a SDK of scbxy board and use it in any Linux Distributions.
```
./kas-container shell kas/rockchip/scbxy-rel.yml -c "bitbake core-image-minimal -c populate_sdk"
```
### Build .deb Install Package for 3rd Part Software
If you want add the software which didn't included in the rootfs by default, You can build the .deb packages for them. And install the .deb file by apt command. Eg.
#### Build DPDK and VPP
```
./kas-container shell kas/rockchip/scbxy-dev.yml -c "bitbake vpp-core"
```
*Notes: vpp-core depends on dpdk, so build vpp-core built create install package for dpdk automatically.*

