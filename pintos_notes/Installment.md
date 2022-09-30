# Installation
The whole project works on M1 Macbook virtual lab machine, this instruction maybe less helpful for x86 users.

 ```shell
 # download qemu and pintos
 $ brew install qemu
 $ zcat pintos.tar.gz | tar x
 ```
 ### Environment Configuration
 ```shell
 $ brew tap nativeos/i386-elf-toolchain

 $ brew install i386-elf-binutils i386-elf-gcc
 
 $ cd /opt/homebrew/bin/qemu-system-aarch64

 $ sudo ln -s qemu-system-aarch64 qemu
 
 $ cd $PINT_OS_PATH/src/threads

 $ make

 $ ../utils/pintos --qemu --run alarm-multiple
 ```
### MacOS Clang Compile Issue

Clang still cannot support some command that runs on gcc. For this project, we need to switch the current compiler to gcc to run it

This [instruction](https://stackoverflow.com/questions/24380456/how-can-i-make-cmake-use-gcc-instead-of-clang-on-mac-os-x) would be helpful if you want to do that