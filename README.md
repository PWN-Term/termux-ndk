# UNFINISHED

This is Google's standard NDK, which only supports running on android devices with aarch64 architecture

the source code from AOSP llvm-toolchain master branch, because llvm is cross-platform, so we can recompile it to Android

At first, we don‘t need to rebuild the whole NDK, since google already built most of it.
we only need to build llvm toolchain, then replace the llvm in the NDK.
of course you can build the whole NDK, use checkbuild.py, but the source code is too huge.

For more details information, please refer to [toolchain readme docs](https://github.com/Lzhiyong/termux-ndk/tree/master/docs)

##### [download r22b](https://github.com/Lzhiyong/termux-ndk/releases)
####  Setting up arch arm64 / android mix env to build in pc

#### Installing req
```
$ apt install qemu-user qemu-user-static
```

#### Setting up arch rootfs
1) download http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
2) mkdir arch-arm64
3) tar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C arch-arm64

##### Some post things
* rm -rf arch-arm64/etc/resolv*
* cp -rf /etc/resolv.conf arch-arm64/etc/resolv.conf
* nano arch-arm64/etc/pacman.conf
* And now comment out CheckSpace and uncomment Color
```
$ cat <<EOF > /bin/ch
#!/bin/bash
FS=$1

echo "Mounting proc"
mount -t proc proc $FS/proc/
sleep 2
echo "Mounting sysfs"
mount -t sysfs sys $FS/sys/
sleep 2
echo "Binding dev"
mount -o bind /dev $FS/dev/
sleep 2
echo "[+] Started chroot"
chroot $FS/ /bin/bash
sleep 2
echo "UMount proc"
umount -l $FS/proc/
sleep 2
echo "UMount sysfs"
umount -l $FS/sys/
sleep 2
echo "Umount dev"
umount -l $FS/dev/
sleep 1

EOF

$ chmod +x /bin/ch

$ ch arch-arm64
```
* In chroot run these
```
$ pacman-key --init
$ pacman-key --populate archlinuxarm
$ pacman -Syu
$ pacman -S patch rsync
```
* To exit chroot run cmd ( exit ), so script can umount bindings
##### Adding android env to arch
* In arm64 phone tar these and then add these to chroot, or download premade arm64 tar's
* https://github.com/PWN-Term/hacky-stuff/releases/tag/28.05.2021
```
$ cd /system && tar -cJf /sdcard/system.tar.xz *

$ cd /apex && tar -cJf /sdcard/apex.tar.xz *

$ cd /vendor && tar -cJf /sdcard/vendor.tar.xz *
```

####  How to build

Termux needs to install aarch64 version of Linux,
I recommend using [TermuxArch](https://github.com/SDRausty/TermuxArch) 
ArchLinux only downloads source code, we are not using it to compile

In order to save storage , there are some pre-compiled tools that do not need to be downloaded. 
comment out them in the llvm-toolchain/.repo/manifests/default.xml file, click [here](https://github.com/Lzhiyong/termux-ndk/blob/master/patches/default.xml.patch) for example.

```bash
# I assume that you have installed the ArchLinux
# install repo
pacman -S repo 

cd /data/data/com.termux/files/home 
mkdir llvm-toolchain && cd llvm-toolchain

# download the source code
repo init -u https://android.googlesource.com/platform/manifest -b llvm-toolchain

# for china
repo init -u https://aosp.tuna.tsinghua.edu.cn/platform/manifest -b llvm-toolchain

repo sync -c -j4

# sync finish, exit ArchLinux
exit

```
 ****

Termux needs to install some build-essential packages, then copy or soft link it to llvm-toolchain/prebuilts

```bash

# remove clang-bootstrap 
rm -vrf llvm-toolchain/prebuilts/clang/host/linux-x86/clang-bootstrap

# extract android-ndk-r22b.tar.xz to /path/to/clang-bootstrap
tar -xJvf android-ndk-r22b.tar.xz -C llvm-toolchain/prebuilts/clang/host/linux-x86/clang-bootstrap

# soft link cmake to llvm-toolchain/prebuilts
ln -sf /data/data/com.termux/files/usr/bin/cmake llvm-toolchain/prebuilts/cmake/linux-x86/bin/cmake

# soft link make to llvm-toolchain/prebuilts
ln -sf /data/data/com.termux/files/usr/bin/make llvm-toolchain/prebuilts/build-tools/linux-x86/bin/make

# soft link ninja to llvm-toolchain/prebuilts
ln -sf /data/data/com.termux/files/usr/bin/ninja llvm-toolchain/prebuilts/build-tools/linux-x86/bin/ninja

# remove prebuilt python
rm -vrf llvm-toolchain/prebuilts/python/linux-x86/*
# apt download python3
# extract python_3.9.x_aarch64.deb to llvm-toolchain/prebuilts/python/linux-x86

# building golang
apt install golang
cd llvm-toolchain/prebuilts/go/linux-x86/src
./make.bash

# copy patches/crypt.h to llvm-toolchain/prebuilts/clang/host/linux-x86/clang-bootstrap/sysroot/usr/include

# copy patches/sFile.h to llvm-toolchain/toolchain/llvm-project/lldb/include/lldb/Utility
# vim llvm-toolchain/toolchain/llvm-project/lldb/include/lldb/Utility/ReproducerInstrumentation.h
# add #include "lldb/Utility/sFile.h"

```

 **** 
####  Building start!

```bash
# no build for windows
# If it is not the first time to build, you can add options --skip-source-setup to save time
# cd back to llvm-toolchain
python toolchain/llvm_android/build.py --enable-assertions --no-build windows
```

llvm-toolchain stage1 and stage2 compilation will take a long time.

or you can compile it on your computer, the operation process is the same.

there may be some errors during the compilation process, please solve it yourself!

 **** 
 
#### Building finish!
```bash
# test the ndk clang
# Note: aarch64-linux-android$API-clang++ is recommended, instead of clang alone
# armv7a-linux-android$API-clang
# i686-linux-android$API-clang
# x86_64-linux-android$API-clang
NDK_TOOLCHAIN=/path/to/android-ndk-r22b/toolchains/llvm/prebuilt/linux-aarch64
$NDK_TOOLCHAIN/bin/aarch64-linux-android30-clang++ test.cpp -o test

# if you want to use clang alone, you need to specify --sysroot=/path/to/sysroot
# for example --sysroot=/data/data/com.termux/files (termux default sysroot)
# if you don't specify a target, the default is --target=aarch64-unknown-linux-android
$NDK_TOOLCHAIN/bin/clang --sysroot=/path/to/sysroot hello.c -o hello

# c++ needs link flags -lc++_shared -static-libstdc++
$NDK_TOOLCHAIN/bin/clang++ --sysroot=/path/to/sysroot -lc++_shared -static-libstdc++ hello.cpp -o hello
```
 **** 
 
#### Building simpleperf
```bash
# simpleperf no need to compile
cd android-ndk-r22b/simpleperf/bin/linux/aarch64

ln -sf ../../android/arm64/simpleperf ./simpleperf

```

 **** 
#### Building shader-tools
```bash
cd /data/data/com.termux/files/home
git clone https://github.com/google/shaderc
cd shaderc/third_party
git clone https://github.com/KhronosGroup/SPIRV-Tools.git   spirv-tools
git clone https://github.com/KhronosGroup/SPIRV-Headers.git spirv-tools/external/spirv-headers
git clone https://github.com/google/googletest.git
git clone https://github.com/google/effcee.git
git clone https://github.com/google/re2.git
git clone https://github.com/KhronosGroup/glslang.git

# start building shaderc...

cd ~/shaderc && mkdir build && cd build

# setting android ndk toolchain
TOOLCHAIN=/path/to/android-ndk-r22b/toolchains/llvm/prebuilt/linux-aarch64

cmake -G "Unix Makefiles" \
    -DCMAKE_C_COMPILER=$TOOLCHAIN/bin/aarch64-linux-android30-clang \
    -DCMAKE_CXX_COMPILER=$TOOLCHAIN/bin/aarch64-linux-android30-clang++ \
    -DCMAKE_SYSROOT=$TOOLCHAIN/sysroot \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_INSTALL_PREFIX=/path/to/shader-tools \
    ..

make -j16
```

 **** 
#### Building renderscript 

```bash
# llvm-rs-cc and bcc_compat
# slang source code
# git clone https://android.googlesource.com/platform/frameworks/compile/slang
# libbcc source code
# git clone https://android.googlesource.com/platform/frameworks/compile/libbcc
# rs souce code
# git clone https://android.googlesource.com/platform/frameworks/rs
# llvm and clang source code
# git clone https://android.googlesource.com/platform/external/llvm
# git clone https://android.googlesource.com/platform/external/clang

cd termux-ndk/renderscript
./build.sh

```
 **** 

#### Building app
**building android app with termux-ndk, please refer to [build-app](https://github.com/Lzhiyong/termux-ndk/tree/master/build-app)**

## Issues

* each NDK version has a lot of changes, direct compilation will fail, 
the build script are not generic, so please solve the error yourself.

* llvm-rs-cc may have bugs, I rewrote the rs_cc_options.cpp file.

* because RSCCOptions.inc compilation error, I can't solve it yet.

* RSCCOptions.inc is generated by llvm-tblgen, but which has errors

* llvm-tblgen -I=../llvm/include RSCCOptions.tb -o RSCCOptions.inc

* if anyone konws, please submit to issue
