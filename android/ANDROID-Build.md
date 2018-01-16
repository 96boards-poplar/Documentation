# Poplar Android Build

The following instructions are provided as guidence to download, build, and install Android on the Poplar board.

For general set up, refer to [official Android doc](https://source.android.com/source/initializing).

## Compiling userspace

1. Get AOSP android-8.0.0_r17
```
mkdir -p ~/poplar
cd ~/poplar
repo init -u https://android.googlesource.com/platform/manifest.git -b android-8.0.0_r17
repo sync -j8
```

2. Add poplar device

```
mkdir device/hisilicon
git clone https://github.com/96boards-poplar/poplar-device.git device/hisilicon/poplar
git clone https://github.com/96boards-poplar/poplar-kernel.git device/hisilicon/poplar-kernel
```

3. Download vendor blobs

- For public

```
wget https://github.com/96boards-poplar/vendor/raw/master/hisilicon-poplar-20180116-b2149740.tgz
tar zxvf hisilicon-poplar-20180116-b2149740.tgz
./extract-hisilicon-poplar.sh
```

Review the license and type "I ACCEPT". The vendor blobs will be extracted as 'vendor' directory under $ANDROID_BUILD_TOP.

- For internal development (ping bin.chen at linaro.org for repo address)

```
git clone vendor_dev.git vendor
```

4. Build
```
source build/envsetup.sh
lunch poplar-eng
make -j8
```


## Installing initial bootloader and partition table

see [Installing initial bootloader and partition table](ANDROID-Flash.md#installing-initial-bootloader-and-partition-table)

## Flashing Android images

see [Flashing Android Images](ANDROID-Flash.md#flashing-android-images)


## Building the kernel

1. Download the necessary toolchain

Download a 64-bit GCC 4.9 toolchain from Linaro, and extract
it under the /opt directory (or anywhere you prefer) on your build system:

```shell
cd /tmp
wget https://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/aarch64-linux-gnu/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
sudo mkdir -p /opt
sudo tar -C /opt -xJf /tmp/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
rm /tmp/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
```

And, set up the `CROSS_64` enviroment variable that will be used in building kernel below.

```shell
CROSS_64=/opt/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```

2. Download and build Poplar kernel:

```
git clone https://github.com/96boards-poplar/linux.git
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

## Licence

[![Creative Commons Licence](https://licensebuttons.net/l/by-sa/4.0/88x31.png)] (http://creativecommons.org/licenses/by-sa/4.0/)
