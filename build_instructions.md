# Building Poplar System Recovery Media From Source

[![Creative Commons Licence](https://licensebuttons.net/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)

The instructions that follow describe the process for creating a USB
flash drive suitable for use in recovering a Poplar system from a
"bricked" state.  The USB memory stick must be at least 2 GB.

## Gather required sources

First you'll gather the source code and other materials required to
package a USB recovery device. These instructions assume you are using Linux based OS on your host machine.

### Step 1: Make sure you have needed tools installed

  This list may well grow, but at least you'll need the following:

```shell
      sudo apt-get update
      sudo apt-get upgrade
      sudo apt-get install device-tree-compiler libssl-dev u-boot-tools
      sudo apt-get install screen simg2img
```

### Step 2: Set up the working directory.

```shell
  mkdir -p ~/src/poplar
  cd ~/src/poplar
  TOP=$(pwd)
```

### Step 3: Download a root file system image to use.
  These are available from Linaro, under here:
    http://snapshots.linaro.org/debian/images/stretch/developer-arm64/
  These images change regularly, and the latest version is always
  available under a folder named "latest".  For the purposes of this
  document we assume that build 80 is used.  If you download this
  file by some means other than "wget" shown below, please ensure it
  gets place in the "recovery" directory created here.

```shell
    mkdir ${TOP}/recovery
    wget -P ${TOP}/recovery \
        http://snapshots.linaro.org/debian/images/stretch/developer-arm64/80/linaro-stretch-developer-20170914-80.tar.gz
```

### Step 4: Get the source code.

```shell
  cd ${TOP}
  git clone https://github.com/ARM-software/arm-trusted-firmware
  git clone https://github.com/96boards-poplar/poplar-tools.git
  git clone https://github.com/96boards-poplar/l-loader.git
  git clone https://github.com/96boards-poplar/u-boot.git
  git clone https://github.com/96boards-poplar/linux.git -b poplar-4.9
  git clone https://github.com/OP-TEE/optee_os
  git clone https://github.com/OP-TEE/optee_client
  git clone https://github.com/OP-TEE/optee_test
```

### Step 5: Set up toolchains for building
  Almost everything uses aarch64, but one item (l-loader.bin) must
  be built for 32-bit ARM.  In addition, UEFI requires the compiler
  to be no newer than GCC 4.9.  Finally, the newest versions of GCC
  warn and in some case treat as errors some things that have not been
  fixed in code that's not Poplar-specific in the Linux kernel code.
  The easiest way to avoid problems due to these requirements is to
  just use the 4.9 version of GCC for 64-bit ARM, and a stable but
  new version of GCC for 32-bit ARM.  If you already have installed
  cross-compilers, they may work for you; but the instructions
  assume you are installing the following toolchains.

  Download a 64-bit toolchain using GCC 4.9 from Linaro, and extract
  it under the /opt directory on your build system:
```shell
    cd /tmp
    wget https://releases.linaro.org/components/toolchain/binaries/4.9-2017.01/aarch64-linux-gnu/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
    sudo mkdir -p /opt
    sudo tar -C /opt -xJf /tmp/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
    rm /tmp/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu.tar.xz
```

  Download a recent 32-bit GCC toolchain from Linaro, and extract it
  under the /opt directory on your build system:
```shell
    cd /tmp
    wget https://releases.linaro.org/components/toolchain/binaries/7.1-2017.08/arm-linux-gnueabi/gcc-linaro-7.1.1-2017.08-x86_64_arm-linux-gnueabi.tar.xz
    sudo tar -C /opt -xJf /tmp/gcc-linaro-7.1.1-2017.08-x86_64_arm-linux-gnueabi.tar.xz
    rm /tmp/gcc-linaro-7.1.1-2017.08-x86_64_arm-linux-gnueabi.tar.xz
```

  Finally, set some environment variables used to specify the path
  (and file name prefix) for accessing the 32-bit and 64-bit cross
  compiler tool chains in the instructions that follow:

```shell
    CROSS_32=/opt/gcc-linaro-7.1.1-2017.08-x86_64_arm-linux-gnueabi/bin/arm-linux-gnueabi-
    CROSS_64=/opt/gcc-linaro-4.9.4-2017.01-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
```

## Build everything

### Step 1: Build U-Boot.
  The result of this process will be a file "u-boot.bin" that will
  be incorporated into a FIP file created for ARM Trusted Firmware code.

```shell
    # This produces one output file, which is used when building ARM
    # Trusted Firmware, next:
    #       u-boot.bin
    cd ${TOP}/u-boot
    make distclean
    make CROSS_COMPILE=${CROSS_64} poplar_defconfig
    make CROSS_COMPILE=${CROSS_64}
```

### Step 2: Build OP-TEE OS.
  The result of this process will be a file "tee-pager.bin" that will
  be incorporated into a FIP file created for ARM Trusted Firmware code.

```shell
    # This produces one output file, which is used when building ARM
    # Trusted Firmware, next:
    #       tee-pager.bin
    cd ${TOP}/optee_os
    rm -rf out

    make PLATFORM=poplar CFG_ARM64_core=y CROSS_COMPILE=${CROSS_32} \
	CROSS_COMPILE_core=${CROSS_64} CROSS_COMPILE_ta_arm64=${CROSS_64} \
	CROSS_COMPILE_ta_arm32=${CROSS_32} CFG_TEE_CORE_LOG_LEVEL=2

    # If using 1GB board
    make PLATFORM=poplar CFG_ARM64_core=y CROSS_COMPILE=${CROSS_32} \
	CROSS_COMPILE_core=${CROSS_64} CROSS_COMPILE_ta_arm64=${CROSS_64} \
	CROSS_COMPILE_ta_arm32=${CROSS_32} CFG_TEE_CORE_LOG_LEVEL=2 \
	CFG_DRAM_SIZE_GB=1
```

### Step 3: Build ARM Trusted Firmware components.
  The result of this process will be two files "bl1.bin" and
  "fip.bin", which will be incorporated into the image created for
  "l-loader" in the next step. The FIP file packages files "bl2.bin"
  and "bl31.bin" (built here) along with "u-boot.bin" and "tee-pager.bin"
  (built earlier).

```shell
    # This produces two output files, which are used when building
    # "l-loader", next:
    #       build/poplar/debug/bl1.bin
    #       build/poplar/debug/fip.bin
    cd ${TOP}/arm-trusted-firmware
    make distclean
    make CROSS_COMPILE=${CROSS_64} all fip DEBUG=1 PLAT=poplar SPD=opteed \
	BL33=${TOP}/u-boot/u-boot.bin \
	BL32=${TOP}/optee_os/out/arm-plat-poplar/core/tee-pager.bin
```

### Step 4: Build "l-loader"
  This requires the two ARM Trusted Firmware components you built
  earlier.  So start by copying them into the "atf" directory.  Note
  that "l-loader" is a 32-bit executable, so you need to use a
  different tool chain.

```shell
    # This produces one output file, which is used in building the
    # USB flash drive:
    #       l-loader.bin
    cd ${TOP}/l-loader
    cp ${TOP}/arm-trusted-firmware/build/poplar/debug/bl1.bin atf/
    cp ${TOP}/arm-trusted-firmware/build/poplar/debug/fip.bin atf/
    make clean
    make CROSS_COMPILE=${CROSS_32}
```

### Step 5: Build Linux.
  The result of this process will be two files: "Image" contains the
  kernel image; and "hi3798cv200-poplar.dtb" containing the
  flattened device tree file (device tree binary).  A Linux build is
  sped up considerably by running "make" with multiple concurrent
  jobs.  JOBCOUNT is set below to something reasonable to benefit
  from this.

```shell
    # This produces two output files, which are used when building
    # the USB flash drive image:
    #       arch/arm64/boot/Image
    #       arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dtb
    cd ${TOP}/linux
    JOBCOUNT=$(grep ^processor /proc/cpuinfo | wc -l)
    make ARCH=arm64 distclean
    make ARCH=arm64 CROSS_COMPILE="${CROSS_64}" poplar_defconfig
    make ARCH=arm64 CROSS_COMPILE="${CROSS_64}" all dtbs modules -j ${JOBCOUNT}
```

### Step 6: Gather the required components you built above
  First gather the files that will be required to create the Poplar
  USB drive recovery image.  The root file system image should
  already have been placed in the "recovery" directory.

```shell
    cd ${TOP}/recovery
    cp ${TOP}/poplar-tools/poplar_recovery_builder.sh .
    cp ${TOP}/l-loader/l-loader.bin .
    cp ${TOP}/linux/arch/arm64/boot/Image .
    cp ${TOP}/linux/arch/arm64/boot/dts/hisilicon/hi3798cv200-poplar.dtb .
```

### Step 7: Build image files used for recovery and installation
  You need to supply the root file system image you downloaded
  earlier (whose name may be different from what's shown below).
  This will also require superuser privilege to complete.

```shell
    # This produces a directory "recovery_files".  In that directory,
    # "fastboot.bin" can be placed on a USB flash drive (formatted
    # with an MBR with a single FAT32 file system partition) that can
    # be used to boot a Poplar board in a "bricked" state.  The
    # remaining files are used in populating the eMMC media with a
    # bootable Linux system.
    #       recovery_files/fastboot.bin
    #       recovery_files/install.scr
    #       recovery_files/install-*.scr
    #       recovery_files/partition1-1of1.gz
    #		...
    bash ./poplar_recovery_builder.sh all \
		    linaro-stretch-developer-20170914-80.tar.gz
```

**NOTE**: If you are running Ubuntu 14.04, sfdisk needs to be upgraded
to a newer version like 2.26.2 in following steps.

```shell
wget https://www.kernel.org/pub/linux/utils/util-linux/v2.26/util-linux-2.26.2.tar.xz
tar xf util-linux-2.26.2.tar.xz
cd util-linux-2.26.2
./configure
make sfdisk
export PATH=/path/to/util-linux-2.26.2:$PATH
```

**NOTE**: If you get below error, that means your `mkimage` version is
too old, e.g. if you using Ubuntu 14.04. Use the one you just built in
`${TOP}/u-boot/tools` instead.

```shell
Invalid CPU Type - valid names are: alpha, arm, x86, ia64, m68k, microblaze, mips, mips64, nios2, powerpc, ppc, s390, sh, sparc, sparc64, blackfin, avr32, nds32, or1k, sandbox
Usage: mkimage -l image
          -l ==> list image header information
       mkimage [-x] -A arch -O os -T type -C comp -a addr -e ep -n name -d data_file[:data_file...] image
          -A ==> set architecture to 'arch'
          -O ==> set operating system to 'os'
          -T ==> set image type to 'type'
          -C ==> set compression type 'comp'
          -a ==> set load address to 'addr' (hex)
          -e ==> set entry point to 'ep' (hex)
          -n ==> set image name to 'name'
          -d ==> use image data from 'datafile'
          -x ==> set XIP (execute in place)
       mkimage [-D dtc_options] [-f fit-image.its|-F] fit-image
          -D => set options for device tree compiler
          -f => input filename for FIT source
Signing / verified boot not supported (CONFIG_FIT_SIGNATURE undefined)
       mkimage -V ==> print version information and exit
```

**NOTE**: It is normal to see below warning during this step.

```shell
recovery_files/partition3 is not a block special device.
Proceed anyway? (y,n)
```

### Step 8: Copy image files to the TFTP home directory
  The recovery process depends on transferring files to the Poplar
  board via Ethernet using TFTP.  The "recovery_files" directory
  must be copied to the root of the TFTP directory.

```shell
    cd ${TOP}/recovery
    sudo rm -rf ~tftp/recovery_files
    sudo cp -a recovery_files ~tftp
    sudo chown -R tftp.tftp ~tftp/recovery_files
```

## To allow recovery of a Poplar board in a "bricked" state, prepare a USB flash drive.

### Step 1: Identify your USB flash drive device

  Insert the USB flash drive into your host system, and identify
  your USB device:

```shell
	grep . /sys/class/block/sd?/device/model
```
  If you recognize the model name as your USB flash device, then
  you know which "sd" device to use.  Here's an example:

```shell
	/sys/class/block/sdh/device/model:Patriot Memory
	                 ^^^
```
  I had a Patriot Memory USB flash drive, and the device name
  I'll want is "/dev/sdh" (based on "sdh" above).  Record this name:

```shell
	USBDISK=/dev/sdh	# Make sure this is *your* device
```

  The instructions that follow assume your USB flash drive needs to be
  formatted "from scratch."  Once formatted, all that's required is to
  copy "fastboot.bin" to the first partition on the drive, and then
  properly eject the medium before removing the USB drive.

### Step 2: Format the flash drive using MBR partitioning.

  THIS IS VERY IMPORTANT.  The following commands will COMPLETELY
  ERASE the contents of whatever device you specify here.  So be
  sure USBDISK defines the flash device you intend to erase.

  You will need superuser access.  First, unmount anything mounted
  on that device:

```shell
    sudo umount ${USBDISK}?
```

  Next, clobber any existing partitioning information that might be
  found at the beginning of the device:

```shell
    sudo dd if=/dev/zero of=${USBDISK} bs=2M count=1 status=none
```

  Create a DOS MBR partition table on the USB flash drive with a
  single partition, and format that partition using FAT32.
```shell
    {   echo label:dos
	echo 1: start=8 size=62496KiB type=0x0c
	echo write
    } | sudo sfdisk --label dos ${USBDISK}
    sudo mkfs.fat -F 32 ${USBDISK}1
```

### Step 3: Copy "fastboot.bin" to the drive

  Finally, mount that partition and copy "fastboot.bin" into it.
  Once the partition has been unmounted and the device has been
  ejected, the USB stick can be removed.

```shell
    cd ${TOP}/recovery/recovery_files
    mkdir -p /tmp/usbdisk
    sudo mount -t vfat ${USBDISK}1 /tmp/usbdisk

    sudo cp fastboot.bin /tmp/usbdisk

    sudo umount /tmp/usbdisk
    rmdir /tmp/usbdisk
    sudo eject ${USBDISK}
```

  (For a previously-formatted drive, simply inserting it will cause
  it be mounted automatically--normally under /media/...  somewhere.)

  Remove the USB flash drive from your host system.

## De-brick a Poplar board in a "bricked" state

  If a Poplar board is in a "bricked" state, it can be booted using
  the USB flash drive prepared above.

### Step 1: Prepare the Poplar board for power-on

- The Poplar board should be powered off.  You should have a cable
  from the Poplar's micro USB based serial port to your host
  system so you can connect and observe activity on the serial port.
  For me, the board console shows up as /dev/ttyUSB0 when the USB
  cable is connected.  The serial port runs at 115200 baud.  I use
  this command to see what's on the console:

```shell
      screen /dev/ttyUSB0 115200
```

### Step 2: Insert the USB flash drive on the Poplar board

- There are a total of 4 USB connectors on the Poplar board.  Two
  are USB 2.0 ports, they're stacked on top of each other.  Insert
  the USB memory stick into one of these two.

- There is a "USB_BOOT" button on the board.  It is one of two
  buttons on same side of the boards as the stacked USB 2.0 ports.
  To boot from the memory stick, this button needs to be depressed
  at power-on.  You only need to hold it for about a second;
  keeping it down a bit longer does no harm.

- Next you will be powering on the board, but you need to interrupt
  the automated boot process.  To do this, be prepared to press a
  key, perhaps repeatedly, in the serial console window until you
  find the boot process has stopped.

### Step 2: Boot the Poplar board from the USB flash drive

- Power on the Poplar board (while pressing the USB_BOOT button),
  and interrupt its automated boot with a key press.  This should
  lead to a "poplar# " prompt.

## Re-flash images onto the Poplar board eMMC

  The files required for partitioning and re-flashing the content of
  eMMC media on the Poplar board were produced earlier, and should
  now be present in ~tftp/recovery_files.  The Ethernet interface on
  the Poplar board must be configured, and then an installer script
  will be downloaded and executed.

### Step 1: Configure the Poplar Ethernet interface

  The following assumes you know your network configuration, and that
  you have an IP address in that network to use for the Poplar board.

- Inform U-Boot about the network parameters to use.  Use values for
  the following environment variables that are appropriate for your
  network.  The IP address for the Poplar board is assigned with
  "ipaddr"; the netmask for the network is defined by "netmask"; and
  the IP address of the TFTP server containing the recovery files
  (probably your development/build machine) is "serverip".

  Enter the following commands in the Poplar serial console to
  configure the Ethernet interface.
```shell
    env set ipaddr 192.168.0.2
    env set netmask 255.255.255.0
    env set serverip 192.168.0.1
```

- Verify your network connection is operational.
```shell
    ping ${serverip}
```

### Step 1: Run the installer

- Load an install script using TFTP, and run it.
```shell
    tftp ${scriptaddr} recovery_files/install.scr
    source ${scriptaddr}
```
  It will take about 5-10 minutes to complete writing out the contents
  of the disk.  The result should look a bit like this:

```shell
    ---------------------------
    | poplar# source ${scriptaddr}
    | ## Executing script at 32000000
    | ETH1: PHY(phyaddr=3, rgmii) link UP: DUPLEX=FULL : SPEED=1000M
    | Using gmac1 device
    | TFTP from server 172.22.22.5; our IP address is 172.22.22.154
    | Filename 'recovery_files/install-layout.scr'.
    | Load address: 0x7800000
    | Loading: #
    |          359.4 KiB/s
    | done
    | Bytes transferred = 368 (170 hex)
    | ## Executing script at 07800000
    | ETH1: PHY(phyaddr=3, rgmii) link UP: DUPLEX=FULL : SPEED=1000M
    | Using gmac1 device
    | TFTP from server 172.22.22.5; our IP address is 172.22.22.154
    | Filename 'recovery_files/mbr.bin.gz'.
    | Load address: 0x8000000
    | Loading: #
    |          98.6 KiB/s
    | done
    | Bytes transferred = 101 (65 hex)
    | Uncompressed size: 512 = 0x200
    |
    | MMC write: dev # 0, block # 0, count 1 ... 1 blocks written: OK
    |          . . .
```

- When this process completes, reset your Poplar board.  You can reset it in
  one of three ways: press the reset button; power the board off and on again;
  or run this command in the serial console window:
```shell
    reset
```

  At this point, Linux should automatically boot from the eMMC.

  You have now booted your Poplar board with open source code that you
  have built yourself.

## Additional information about the recovery files

  The following paragraphs provide some more information about the
  files found in the "recovery_files" directory.

#### fastboot.bin
  When this file is placed in the first partition of a USB memory
  stick formatted with a FAT32 file system, that memory stick can
  be used to boot the Poplar board.  This is useful if the board
  has become "bricked" and is otherwise unusable.

#### install, install-layout, install-partition1, install-partitionX
  These are human-readable versions of installer scripts used by
  U-Boot.  The top-level installer is "install"; it loads and
  executes the other install scripts.  Each install script has a
  corresponding ".scr" file (e.g., "install.scr"), which is the file
  that U-Boot actually uses.  "install-layout" installs the Master
  Boot Record and the Extended Boot Records required for partitions
  5 and above.  "install-partitionX" contains commands to install
  the contents of just one partition.

  Each "install*.scr" file can be loaded into U-Boot and run.  If
  the top-level "install.scr" is used, it will execute all the
  others.  Otherwise, partial installs can be performed by, for
  example, loading and running "install-layout.scr" to re-write the
  boot records, or "install-partition2.scr" to re-write only
  partition 2.

#### mbr.bin.gz, ebr5.bin.gz, ebr6.bin.gz
  These are the Master Boot Record and Extended Boot Records for
  partitions 5 and 6.  They are compressed.  They are normally
  loaded and flashed to eMMC using "install-layout".

#### partition1.1-of-1.gz, partition3.1-of-4.gz, etc.
  These are files that contain (parts of) the contents of the
  partitions.  The contents of an entire partition can't fit
  entirely in memory, so large partitions are broken into pieces.
  Each piece is compressed.  The install script for the partition
  takes care of uncompressing each part before writing it to eMMC.

## Testing OP-TEE

### Step 1: Build OP-TEE client

```shell
	cd ${TOP}/optee_client
	rm -rf out
	make CROSS_COMPILE=${CROSS_64}
```

### Step 2: Build OP-TEE test

```shell
	cd ${TOP}/optee_test
	rm -rf out
	make CROSS_COMPILE_HOST=${CROSS_64} CROSS_COMPILE_TA=${CROSS_32} \
	  TA_DEV_KIT_DIR=${TOP}/optee_os/out/arm-plat-poplar/export-ta_arm32 \
	  OPTEE_CLIENT_EXPORT=${TOP}/optee_client/out/export
```

### Step 3: Build OP-TEE Debian package

```shell
	export OPTEE_PKG_VERSION=$(cd ${TOP}/optee_os && git describe)-0

	cd ${TOP}
	mkdir -p debpkg/optee_${OPTEE_PKG_VERSION}/usr/bin
	mkdir -p debpkg/optee_${OPTEE_PKG_VERSION}/usr/lib/aarch64-linux-gnu
	mkdir -p debpkg/optee_${OPTEE_PKG_VERSION}/lib/optee_armtz
	mkdir -p debpkg/optee_${OPTEE_PKG_VERSION}/DEBIAN

	cd ${TOP}/debpkg/optee_${OPTEE_PKG_VERSION}/usr/bin
	cp -f ${TOP}/optee_client/out/export/bin/tee-supplicant .
	cp -f ${TOP}/optee_test/out/xtest/xtest .

	cd ${TOP}/debpkg/optee_${OPTEE_PKG_VERSION}/usr/lib/aarch64-linux-gnu
	cp -f ${TOP}/optee_client/out/export/lib/libtee* .

	cd ${TOP}/debpkg/optee_${OPTEE_PKG_VERSION}/lib/optee_armtz
	find ${TOP}/optee_test/out/ta -name "*.ta" -exec cp {} . \;

	cd ${TOP}/debpkg/optee_${OPTEE_PKG_VERSION}/DEBIAN
	echo "Package: op-tee" > control
	echo "Version: ${OPTEE_PKG_VERSION}" >> control
	echo "Section: base" >> control
	echo "Priority: optional" >> control
	echo "Architecture: arm64" >> control
	echo "Depends:" >> control
	echo "Maintainer: OP-TEE <op-tee@linaro.org>" >> control
	echo "Description: OP-TEE client binaries, test program and Trusted Applications" >> control
	echo " Package contains tee-supplicant, libtee.so, xtest and a set of" >> control
	echo " Trusted Applications." >> control
	echo " NOTE! This package should only be used for testing and development." >> control

	cd ${TOP}/debpkg
	dpkg-deb --build optee_${OPTEE_PKG_VERSION}
	cp -f optee_${OPTEE_PKG_VERSION}.deb ${TOP}/recovery/recovery_files/
```

### Step 4: Install OP-TEE Debian package
Boot board to command prompt.
Run `ifconfig` and note its IP address.

From the PC:

```shell
	cd ${TOP}/recovery/recovery_files
	scp optee_${OPTEE_PKG_VERSION}.deb linaro@<poplar_board_ip>:/tmp
```

From the board:

```shell
	cd /tmp
	dpkg --force-all -i optee*.deb
```

### Step 5. Run OP-TEE test
From the board:

```shell
	tee-supplicant &
	xtest
```

Towards the end of the test, you should see something like:

```shell
	+-----------------------------------------------------
	Result of testsuite regression:
	regression_1001 OK
	regression_1002 OK
	...
	regression_8002 OK
	+-----------------------------------------------------
	15677 subtests of which 0 failed
	68 test cases of which 0 failed
	0 test case was skipped
```
