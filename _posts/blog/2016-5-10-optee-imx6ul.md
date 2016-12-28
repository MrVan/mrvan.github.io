---
layout: post
title: OP-TEE on i.MX6UL
description: How to run OP-TEE on i.MX6UL
category: blog
---

### 1. Get [U-Boot](https://github.com/MrVan/u-boot/commits/imx_v2016.03_4.1.15_2.0.0_ga)

```
Compile:
  make ARCH=arm mx6ul_14x14_evk_optee_defconfig
  make ARCH=arm
Burn u-boot.imx to offset 0x400 of SD card:
  dd if=u-boot.imx of=/dev/sd[x] bs=512 seek=2 conv=sync conv=notrunc
```

### 2. Get [Linux Kernel](https://github.com/MrVan/linux/tree/imx_4.1.15_2.0.0_ga)

Use NXP vendor kernel latest ga release 4.1.15.

#### Compile the Kernel:

```
make ARCH=arm imx_v7_defconfig
make ARCH=arm
```
Copy zImage and imx6ul_14x14_evk.dtb to SD card.

In the dts file, caam is disabled. This will be added in future.

### 3. OP-TEE OS

> Code: https://github.com/OP-TEE/optee_os

How to Compile:

```c
make PLATFORM=imx-mx6ulevk ARCH=arm CFG_PAGEABLE_ADDR=0 CFG_NS_ENTRY_ADDR=0x80800000 CFG_DT_ADDR=0x83000000 CFG_DT=y DEBUG=y CFG_TEE_CORE_LOG_LEVEL=4
mkimage -A arm -O linux -C none -a 0x9c0fffe4 -e 0x9c100000 -d ./out/arm-plat-imx/core/tee.bin uTee
```

Recommend you use the mkimage from uboot/tools.

If do not want to see lots log, change CFG_TEE_CORE_LOG_LEVEL to 0.

### 4. OPTEE CLIENT

> Code: https://github.com/OP-TEE/optee_client

```c
make ARCH=arm

After compilation:
Copy all files in out/export/* to your rootfs root directory in your sd card.
```

### 5. OPTEE XTEST

> Code: https://github.com/OP-TEE/optee_test

```c

Compile, please change the directory to yours:

export TA_DEV_KIT_DIR=/home/Freenix/work/forfun/trustzone/optee_os/out/arm-plat-imx/export-ta_arm32
export OPTEE_CLIENT_EXPORT=/home/Freenix/work/forfun/trustzone/optee_client/out/export
export CROSS_COMPILE_HOST=arm-poky-linux-gnueabi-
export CROSS_COMPILE_TA=arm-poky-linux-gnueabi-
export CROSS_COMPILE=arm-poky-linux-gnueabi-

make ARCH=arm
```

After compilation:

-  Copy all xx.ta in out/* to your sd card rootfs /lib/optee_armtz/
-  Copy xtest to your sd card rootfs /bin

### 6. Boot your optee os:

```c
    run loadfdt;
    run loadimage;
    fatload mmc 1:1 0x84000000 uTee;
    run mmcargs;
    bootz 0x84000000 - ${fdt_addr};

    After linux boots up.
    Run: tee-supplicant &
    Run: xtest
    Now you'll see optee test case run.
    When doing OS feature test, it maybe slow, just be patient.
```

