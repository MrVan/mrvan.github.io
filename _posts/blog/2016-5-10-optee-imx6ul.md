---
layout: post
title: OP-TEE on i.MX6UL
description: How to run OP-TEE on i.MX6UL
category: blog
---

### 1. Get [U-Boot](https://github.com/MrVan/uboot/commit/4f016adae573aaadd7bf6a37f8c58a882b391ae6)

```
make ARCH=arm mx6ul_14x14_evk_optee_defconfig
make ARCH=arm
Burn u-boot.imx to offset 0x400 of SD card
```

### 2. Get [Linux Kernel](https://github.com/linaro-swg/linux/tree/optee)

Use branch optee. We do not use NXP vendor kernel latest ga release 4.1.15.
If want to use NXP 4.1.15 ga release, we may need to switch to 1.0.0 optee
release.

Patch the kernel:

```c
    diff --git a/arch/arm/boot/dts/imx6ul-14x14-evk.dts b/arch/arm/boot/dts/imx6ul-14x14-evk.dts
    index 6aaa5ec..2ac9c80 100644
    --- a/arch/arm/boot/dts/imx6ul-14x14-evk.dts
    +++ b/arch/arm/boot/dts/imx6ul-14x14-evk.dts
    @@ -23,6 +23,13 @@
     		reg = <0x80000000 0x20000000>;
     	};
     
    +	firmware {
    +		optee {
    +			compatible = "linaro,optee-tz";
    +			method = "smc";
    +		};
    +	};
    +
     	regulators {
     		compatible = "simple-bus";
     		#address-cells = <1>;
```

#### Compile the Kernel:

```
make ARCH=arm imx_v6_v7_defconfig
make menuconfig
select the two entries
	CONFIG_TEE=y
	CONFIG_OPTEE
make ARCH=arm
```
Copy zImage and imx6ul_14x14_evk.dtb to SD card.

### 3. OP-TEE OS

> Code: https://github.com/MrVan/optee_os/commits/master

Compile:
```
PLATFORM_FLAVOR=mx6ulevk make ARCH=arm PLATFORM=imx DEBUG=1 CFG_TEE_CORE_LOG_LEVEL=0
arm-poky-linux-gnueabi-objdump -D out/arm-plat-imx/core/tee.elf > tee.s
arm-poky-linux-gnueabi-objcopy -O binary out/arm-plat-imx/core/tee.elf optee.bin
If want to see more log, change CFG_TEE_CORE_LOG_LEVEL to 4.
```

### 4. OPTEE CLIENT
> Code: https://github.com/OP-TEE/optee_client
```
Tested commit:
"
88acd6bda5f9e19124fce0015fe64a6644eff036
Support OP-TEE in generic TEE subsystem
"
Compile:
make ARCH=arm

After compilation:
Copy all files in out/export/* to your rootfs root directory in your sd card.
```

### 5. OPTEE XTEST

> Code: https://github.com/OP-TEE/optee_test
```
Tested commit:
"
commit c8186b363cd3e36946041a9365f2fd423288e227
Author: Zoltan Kuscsik <zoltan.kuscsik@linaro.org>
Date:   Fri Apr 8 11:35:17 2016 +0200

    Use correct Android include paths
"

Compile, please change the directory to yours:

export TA_DEV_KIT_DIR=/home/Freenix/work/forfun/trustzone/optee_os/out/arm-plat-imx/export-ta_arm32
export OPTEE_CLIENT_EXPORT=/home/Freenix/work/forfun/trustzone/optee_client/out/export
export CROSS_COMPILE_HOST=arm-poky-linux-gnueabi-
export CROSS_COMPILE_TA=arm-poky-linux-gnueabi-
export CROSS_COMPILE=arm-poky-linux-gnueabi-

make ARCH=arm
```

After compilation:

> Copy all xx.ta in out/* to your sd card rootfs /lib/optee_armtz/
> Copy xtest to your sd card rootfs /bin

### 6. Boot your optee os:

```
    run loadfdt;
    run loadimage;
    fatload mmc 1:1 0x9c100000 optee.bin;
    run mmcargs;
    bootz ${loadaddr} - ${fdt_addr};

    After linux boots up.
    Run: tee-supplicant &
    Run: xtest
    Now you'll see optee test case run.
    When doing OS feature test, it maybe slow, just be patient.
```

