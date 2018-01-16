# Building Poplar Debian System Recovery Media From Source

[![Creative Commons Licence](https://licensebuttons.net/l/by-sa/4.0/88x31.png)](http://creativecommons.org/licenses/by-sa/4.0/)

The instructions that follow describe the process for creating a USB
flash drive suitable for use in recovering a Poplar system from a
"bricked" state.  The USB memory stick must be at least 2 GB.

**NOTE:** To create recovery media from pre-built images rather than
building from source, follow the [Create a USB drive for flashing](android/ANDROID-Flash.md#create-a-usb-drive-for-flashing) and [Installing initial bootloader and partition table](android/ANDROID-Flash.md#installing-initial-bootloader-and-partition-table) sections of [android/ANDROID-Flash.md](android/ANDROID-Flash.md). After that, proceed to the [De-brick a Poplar board in a "bricked" state](#de-brick-a-poplar-board-in-a-bricked-state) section below.

## Building the image files from source

Follow the [build instructions](debian_build_instructions.md) in
[debian_build_instructions.md](debian_build_instructions.md) to create
image suitable for use in a Poplar system, with the following exceptions:

### Step 1: Build ARM Trusted Firmware components for recovery.
Add `POPLAR_RECOVERY=1` to the end of the `make` command, as shown below:

```shell
    cd ${TOP}/arm-trusted-firmware
    make distclean
    make CROSS_COMPILE=${CROSS_64} all fip DEBUG=1 PLAT=poplar SPD=none \
        BL33=${TOP}/u-boot/u-boot.bin POPLAR_RECOVERY=1
```

### Step 2: Build "l-loader" for recovery
Add `CFLAGS=-DRECOVERY` to the end of the `make` command, as shown below:

```shell
    cp ${TOP}/arm-trusted-firmware/build/poplar/debug/bl1.bin atf/
    cp ${TOP}/arm-trusted-firmware/build/poplar/debug/fip.bin atf/
    make clean
    make CROSS_COMPILE=${CROSS_32} CFLAGS=-DRECOVERY
```

Once you are done with [Step 7](debian_build_instructions.md#step-7-copy-image-files-to-the-tftp-home-directory)
in the [Build everything](debian_build_instructions.md#build-everything)
section, come back to this document.

## To allow recovery of a Poplar board in a "bricked" state, prepare a USB flash drive.

### Step 1: Identify your USB flash drive device

  Insert the USB flash drive into your host system, and identify
  your USB device:

```shell
	grep . /sys/class/block/sd?/device/model
```
  If you recognize the model name as your USB flash device, then
  you know which "sd" device to use.  Here is an example:

```shell
	/sys/class/block/sdh/device/model:Patriot Memory
	                 ^^^
```
  I had a Patriot Memory USB flash drive, and the device name
  I will want is "/dev/sdh" (based on "sdh" above).  Record this name:

```shell
	USBDISK=/dev/sdh	# Make sure this is *your* device
```

  The instructions that follow assume your USB flash drive needs to be
  formatted "from scratch."  Once formatted, all that is required is to
  copy `fastboot.bin` to the first partition on the drive, and then
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

  Finally, mount that partition and copy `fastboot.bin` into it.
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
  it be mounted automatically--normally under `/media/...`  somewhere.)

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
  are USB 2.0 ports, they are stacked on top of each other.  Insert
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

### Step 3: Boot the Poplar board from the USB flash drive

- Power on the Poplar board (while pressing the USB_BOOT button),
  and interrupt its automated boot with a key press.  This should
  lead to a `poplar# ` prompt.

- If the board does not power up properly, something is wrong with the
  images built or USB flash drive created. The console log should give
  some details regarding the error.

## Re-flash images onto the Poplar board eMMC

  Follow [Steps 3-4](debian_build_instructions.md#step-3-configure-the-poplar-ethernet-interface)
of the [Flash images onto the Poplar board eMMC](debian_build_instructions.md#flash-images-onto-the-poplar-board-emmc)
section in [debian_build_instructions.md](debian_build_instructions.md)
to re-flash working images onto the board.

## Additional information about the recovery files

  The following paragraphs provide some more information about the
  files found in the `recovery_files` directory.

#### fastboot.bin
  When this file is placed in the first partition of a USB memory
  stick formatted with a FAT32 file system, that memory stick can
  be used to boot the Poplar board.  This is useful if the board
  has become "bricked" and is otherwise unusable.

#### install, install-layout, install-partition1, install-partitionX
  These are human-readable versions of installer scripts used by
  U-Boot.  The top-level installer is `install`; it loads and
  executes the other install scripts.  Each install script has a
  corresponding ".scr" file (e.g., `install.scr`), which is the file
  that U-Boot actually uses.  `install-layout` installs the Master
  Boot Record and the Extended Boot Records required for partitions
  5 and above.  `install-partitionX` contains commands to install
  the contents of just one partition.

  Each `install*.scr` file can be loaded into U-Boot and run.  If
  the top-level `install.scr` is used, it will execute all the
  others.  Otherwise, partial installs can be performed by, for
  example, loading and running `install-layout.scr` to re-write the
  boot records, or `install-partition2.scr` to re-write only
  partition 2.

#### mbr.bin.gz, ebr5.bin.gz, ebr6.bin.gz
  These are the Master Boot Record and Extended Boot Records for
  partitions 5 and 6.  They are compressed.  They are normally
  loaded and flashed to eMMC using `install-layout`.

#### partition1.1-of-1.gz, partition3.1-of-4.gz, etc.
  These are files that contain (parts of) the contents of the
  partitions.  The contents of an entire partition can't fit
  entirely in memory, so large partitions are broken into pieces.
  Each piece is compressed.  The install script for the partition
  takes care of uncompressing each part before writing it to eMMC.
