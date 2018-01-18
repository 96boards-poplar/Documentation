# Build and Run OP-TEE on Poplar

Before trying to build and run OP-TEE on Poplar, please read [instructions](debian_build_instructions.md)
for general setup of working directory, toolchain and installation.

## Get the source code

```shell
cd ${TOP}
git clone https://github.com/OP-TEE/optee_os
git clone https://github.com/OP-TEE/optee_client
git clone https://github.com/OP-TEE/optee_test
```
## Build OP-TEE OS

The result of this process will be a file "tee-pager.bin" that will
be incorporated into a FIP file created for ARM Trusted Firmware code.

```shell
cd ${TOP}/optee_os
rm -rf out

make PLATFORM=poplar CFG_ARM64_core=y CROSS_COMPILE=${CROSS_32} \
     CROSS_COMPILE_core=${CROSS_64} CROSS_COMPILE_ta_arm64=${CROSS_64} \
     CROSS_COMPILE_ta_arm32=${CROSS_32} CFG_TEE_CORE_LOG_LEVEL=2

# Option CFG_DRAM_SIZE_GB=1 is there for board with 1GB DDR.  For 2GB board,
# simply drop the option.
make PLATFORM=poplar CFG_ARM64_core=y CROSS_COMPILE=${CROSS_32} \
     CROSS_COMPILE_core=${CROSS_64} CROSS_COMPILE_ta_arm64=${CROSS_64} \
     CROSS_COMPILE_ta_arm32=${CROSS_32} CFG_TEE_CORE_LOG_LEVEL=2 \
     CFG_DRAM_SIZE_GB=1
```

## Build ARM Trusted Firmware with OP-TEE OS included

To build ARM Trusted Firmware with OP-TEE OS included, build options "SPD" and
"BL32" need to be set up properly.  The following example builds u-boot and
OP-TEE OS built above into ARM Trusted Firmware.

```shell
cd ${TOP}/arm-trusted-firmware
make distclean
make CROSS_COMPILE=${CROSS_64} all fip DEBUG=1 PLAT=poplar SPD=opteed \
     BL33=${TOP}/u-boot/u-boot.bin \
     BL32=${TOP}/optee_os/out/arm-plat-poplar/core/tee-pager.bin
```

## Test OP-TEE

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
