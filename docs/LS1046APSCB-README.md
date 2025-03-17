# LS1046APSCB Yocto Build Environment (KAS)

## Build Rootfs
```
./kas-container build kas/nxp/ls1046apscb-dev.yml
```
## Build Composite Firmware
```
./kas-container shell kas/nxp/ls1046apscb-dev.yml -c "bitbake secure-boot-qoriq"
./kas-container shell kas/nxp/ls1046apscb-dev.yml -c "bitbake qoriq-composite-firmware"
./kas-container shell kas/nxp/ls1046apscb-dev.yml -c "bitbake generate-boottgz"
```
