# Install U-boot on the STM32MP2 SD card

This is needed for some aspects of Bootlin's kernel, Yocto Project and Buildroot labs,
such as saving U-Boot environment settings to non volatile storage.

Tested on the following board revisions:
- STM32MP257F-DK

## 1. Download the images

We suggest you to clone this repository, where all binary images are
ready to be used. You can also build them on your own if you wish, all
instructions are provided in chapter 3.

```
git clone https://github.com/bootlin/training-materials.git
cd training-materials/lab-data/common/bootloader/stm32mp2/
```

## 2. Flashing

Take a micro-SD card and connect it to your PC:
- Either using a direct SD slot if available.
  In this case, the card should be seen as `/dev/mmcblk0` by
  your computer (check the `dmesg` command output).
- Either using a memory card reader.
  In this case, the card should be seen as `/dev/sdb`, or `/dev/sdc`, etc.

Run the mount command to check for mounted SD card
partitions. Umount them with a command such as
`sudo umount /dev/mmcblk0p*` or `sudo umount /dev/sdb*`,
depending on how the system sees the media card device.

Erase the existing partition table and partition contents by simply
zero-ing the first 128 MiB of the SD card
(we assume that the card is seen as `/dev/mmcblk0`):

    $ sudo dd if=/dev/zero of=/dev/mmcblk0 bs=1M count=128

Let’s use the parted command to create the partitions that we are going to use
(we assume that the card is seen as `/dev/mmcblk0`):

    $ sudo parted /dev/mmcblk0

The ROM monitor handles GPT partition tables, let’s create one:
    
    (parted) mklabel gpt

Then, the 3 partitions are created with:

    (parted) mkpart fsbl1 0% 4095s
    (parted) mkpart fsbl2 4096s 6143s
    (parted) mkpart fip 6144s 14335s

You can verify everything looks right with:

    (parted) print
    Model: SD SD16G (sd/mmc)
    Disk /dev/mmcblk0: 15,5GB
    Sector size (logical/physical): 512B/512B
    Partition Table: gpt
    Disk Flags: 

    Number  Start   End     Size    File system  Name   Flags
    1      1049kB  2097kB  1049kB               fsbla1
    2      2097kB  3146kB  1049kB               fsbla2
    3      3146kB  7340kB  4194kB               fip

    (parted)

Once done, quit:

    (parted) quit

Now type the below command to flash your micro-SD card (we assume that
the card is seen as `/dev/mmcblk0`):

    $ sudo dd if=tf-a-stm32mp257f-dk.stm32 of=/dev/mmcblk0p1 bs=1M conv=fdatasync
    $ sudo dd if=tf-a-stm32mp257f-dk.stm32 of=/dev/mmcblk0p2 bs=1M conv=fdatasync
    $ sudo dd if=fip.bin of=/dev/mmcblk0p3 bs=1M conv=fdatasync


## 3. How images were built

### Installing Toolchain

```
cd ~
wget https://developer.arm.com/-/media/Files/downloads/gnu/14.2.rel1/binrel/arm-gnu-toolchain-14.2.rel1-x86_64-aarch64-none-linux-gnu.tar.xz
export PATH=$PATH:$HOME/Downloads/arm-gnu-toolchain-14.2.rel1-x86_64-aarch64-none-linux-gnu/bin
```

### Compiling U-Boot

```
cd ~
git clone https://github.com/STMicroelectronics/u-boot.git
cd u-boot
git checkout v2023.10-stm32mp
export CROSS_COMPILE=aarch64-none-linux-gnu-
make stm32mp25_defconfig
make DEVICE_TREE=stm32mp257f-dk all
```

### Compiling OP-TEE

```
cd ~
git clone clone https://github.com/STMicroelectronics/optee_os.git
cd optee_os
git checkout 4.0.0-stm32mp
make O=build CROSS_COMPILE=aarch64-none-linux-gnu- \
CROSS_COMPILE_core=aarch64-none-linux-gnu- \
CROSS_COMPILE_ta_arm64=aarch64-none-linux-gnu- \
CFG_ARM64_core=y CFG_USER_TA_TARGETS=ta_arm64 \
PLATFORM=stm32mp2 CFG_WITH_TUI=n all
```

### Compiling TF-A

```
cd ~
git https://github.com/STMicroelectronics/arm-trusted-firmware.git 
cd arm-trusted-firmware
git checkout v2.10-stm32mp
make CROSS_COMPILE=aarch64-none-linux-gnu- \
ARM_ARCH_MAJOR=8 ARCH=aarch64 PLAT=stm32mp2 \
STM32MP_LPDDR4_TYPE=1 STM32MP_SDMMC=1 \
DTB_FILE_NAME=stm32mp257f-dk.dtb \
BL32=../optee_os/build/core/tee-header_v2.bin \
BL32_EXTRA1=../optee_os/build/core/tee-pager_v2.bin \
BL32_EXTRA2=../optee_os/build/core/tee-pageable_v2.bin \
SPD=opteed BL33=../u-boot/u-boot-nodtb.bin \
BL33_CFG=../u-boot/u-boot.dtb fip all 
```
