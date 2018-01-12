# Poplar Android Build

Use the following commands to download, build, and install Android on the Poplar board.

For general set up, refer to [official Android doc](https://source.android.com/source/initializing).

## Compiling userspace

1. Get AOSP android-8.0.0_r17
```
mkdir poplar
cd poplar
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

```
wget https://github.com/96boards-poplar/vendor/raw/master/hisilicon-poplar-00001-0c2dc85e.tgz
tar zxvf hisilicon-poplar-00001-0c2dc85e.tgz
./extract-hisilicon-poplar.sh
```

Review the license and type "I ACCEPT". The vendor blobs will be extracted as 'vendor' directory under $ANDROID_BUILD_TOP.

4. Build
```
source build/envsetup.sh
lunch poplar-eng
make -j8
```

## Installing initial bootloader and partition table

see [this](Android-Flash.md#installing-initial-bootloader-and-partition-table)

## Flashing Android images

see [this](Android-Flash.md#flashing-android-images)


## Building the kernel

1. Donwload toolchain.

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

2. Run the following commands:

```
git clone https://github.com/96boards-poplar/linux.git linux
cd linux
make ARCH=arm64 poplar_defconfig
make ARCH=arm64 CROSS_COMPILE=${CROSS_64} -j8
```

3. Copy output to the poplar prebuild kernel directory

```
POPLAR_PREBUILT_KERNEL=${ANDROID_BUILD_TOP}/device/hisilicon/poplar-kernel
cp ./arch/arm64/boot/Image                      ${POPLAR_PREBUILT_KERNEL}/Image
cp ./arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dtb   ${POPLAR_PREBUILT_KERNEL}/hi3798cv200-poplar.dtb
```

4. Make the boot image:

```
make bootimage -j8
```