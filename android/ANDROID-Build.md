# Poplar Android Build

The following instructions are provided as guidence to download, build, and install Android on the Poplar board.

For general set up, refer to [official Android doc](https://source.android.com/source/initializing).

## Compiling userspace

1. Get AOSP
```
mkdir -p ~/poplar
cd ~/poplar
repo init -u https://android.googlesource.com/platform/manifest.git -b master
repo sync -j8
```

2. Add Poplar device and Pre-built 4.9 kernel/dtb

```
mkdir device/hisilicon
git clone https://github.com/96boards-poplar/poplar-device.git device/hisilicon/poplar
git clone https://github.com/96boards-poplar/poplar-kernel.git device/hisilicon/poplar-kernel
```

3. Build
```
source build/envsetup.sh
lunch poplar-eng
make -j8
```

## Installing partition table and bootloader

see [Installing partition table and bootloader](ANDROID-Flash.md#installing-partition-table-and-bootloader)

## Flashing Android images

1. Put board into fastboot mode.

During boot up, interrupt the normal boot flow and get into the the uboot console, type: 

```
usb reset
fastboot 0

```

2. Flash from the host

Check if device is in fastboot mode using follow command, you should get a fastboot device.

```
$sudo fastboot devices
0123456789POPLAR	fastboot
```

Then, flash using `fastboot flash` command:

```bash
cd `$OUT`
sudo fastboot flash mmcsda2 boot.img
sudo fastboot flash mmcsda3 system.img
sudo fastboot flash mmcsda5 vendor.img
sudo fastboot flash mmcsda6 cache.img
sudo fastboot flash mmcsda7 userdata.img
```

## Building the kernel

1. Download the necessary toolchain

Download a 64-bit GCC toolchain from Linaro, and extract
it under the /opt directory (or anywhere you prefer) on your build system:

```shell
cd /tmp
wget https://releases.linaro.org/components/toolchain/binaries/7.1-2017.08/aarch64-linux-gnu/gcc-linaro-7.1.1-2017.08-x86_64_aarch64-linux-gnu.tar.xz
sudo mkdir -p /opt
sudo tar -C /opt -xJf /tmp/gcc-linaro-7.1.1-2017.08-x86_64_aarch64-linux-gnu.tar.xz
rm /tmp/gcc-linaro-7.1.1-2017.08-x86_64_aarch64-linux-gnu.tar.xz
```

And, set up the `CROSS_64` enviroment variable that will be used in building kernel below.

```shell
CROSS_64=/opt/gcc-linaro-7.1.1-2017.08-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```

2. Download and build Poplar kernel:

```
git clone -b poplar-android-4.9 https://github.com/96boards-poplar/linux.git
cd linux
make ARCH=arm64 poplar_defconfig
make ARCH=arm64 CROSS_COMPILE=${CROSS_64} -j8
```

3. Copy new pre-built kernel and device tree files to the Android tree

```
POPLAR_PREBUILT_KERNEL=${ANDROID_BUILD_TOP}/device/hisilicon/poplar-kernel
cp ./arch/arm64/boot/Image ${POPLAR_PREBUILT_KERNEL}/Image
cp ./arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dtb ${POPLAR_PREBUILT_KERNEL}/hi3798cv200-poplar.dtb
```

4. Make the boot image:

```
make bootimage -j8
```

## Known Issue

1. The boot ROM can't recognize certain type of USB disk, the consequence is you can't use that usb disk for recovery flash. The exactly reason and what type of USB disk can't recognized isn't clear at the moment.

2. No serial output if the device is rebooted by power off and on when both USB2 and MicroUSB are connected at the same time. You have to unplug the USB2 to make sure the board completely lose power before rebooting and then reconnect the USB2 cable. An alternative, actually the recommend way is to reboot the device using software reboot command, that is `reboot` in linux console or `reset` in u-boot console.
