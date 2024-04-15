# LS1046APSCB Yocto Build Environment (KAS)

## Build Rootfs
```
kas build kas/nxp/ls1046apscb-dev.yml
```
## Build Composite Firmware
```
kas shell kas/nxp/ls1046apscb-dev.yml -c "bitbake secure-boot-qoriq"
```
