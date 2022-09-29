# Installation
The whole project works on UMN virtual lab machine (Ubuntu x86), if you are implementing on your personal computer, please install the essential dependency

 ```shell
 # download bochs and pintos
 $ tar zxvf bochs-2.6.tar.gz
 $ zcat pintos.tar.gz | tar x
 ```
 ### Environment Configuration
 ```shell
# dependency needed
 $ sudo apt-get install buid-essential

 $ sudo apt-get install xorg-dev

 $ sudo apt-get install bison

 $ sudo apt-get install libgtk2.0-dev

 $ sudo apt-get install libc6:i386 libgcc1:i386 gcc-4.6-base:i386 libstdc++5:i386 libstdc++6:i386 

 $ sudo apt-get install libncurses5:i386

 $ sudo apt-get install g++-multilib

 $ cd bochs-2.6

 $ ./configure --enable-gdb-stub

 $ make

 $ sudo make install
 ```
