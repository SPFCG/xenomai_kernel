# [HOW TO] Built Raspbian Linux kernel with xenomai 3.0.5 path for Rasberry Pi 3 on Ubuntu


## 0. Prepare environment
---
```shell
# install git and subversion
sudo apt-get update
sudo apt-get install git subversion

# Preapre variables
export BUILDDIR=/tmp/rpi3
# build kernel with NUMCORES
export NUMCORES=4

# directory where download and build everything  
mkdir $BUILDDIR   
#  directory for ready to install files
mkdir  $BUILDDIR/dist  
```

## 1. Download sources 
---
**Go to $BUILDDIR directory:**
```shell
cd $BUILDDIR
```
### 1a. Download Raspbian Linux 4.1.y source

```shell
git clone https://github.com/raspberrypi/linux.git
cd  linux/
git  checkout  rpi-4.1.y
cd ..
```
**alternative and faster version:**
```shell
svn export https://github.com/SPFCG/xenomai_kernel/trunk/mirror/rpi-4.1.y/linux linux
```

### 1b. Download Xenomai 3.0.5 source
```shell
wget http://git.xenomai.org/xenomai-3.git/snapshot/xenomai-3-3.0.5.tar.bz2
tar -xjvf xenomai-3-3.0.5.tar.bz2
```
**alternative version:**
```shell
svn export https://github.com/SPFCG/xenomai_kernel/trunk/mirror/xenomai-3-3.0.5 xenomai-3-3.0.5
```

### 1c. Download ipipe patched ipipe with small trivial fix for kernel configuration on bcm-2709  (effects both kernel and some drivers(using interrupts))
```shell
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/ipipe-core-4.1.18-arm-10.fixed.patch
```

**original version:**
```shell
wget https://xenomai.org/downloads/ipipe/v4.x/arm/ipipe-core-4.1.18-arm-10.patch
# older:
wget https://xenomai.org/downloads/ipipe/v4.x/arm/older/ipipe-core-4.1.18-arm-9.patch


# mirrors:

wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/ipipe-core-4.1.18-arm-10.patch
# older
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/ipipe-core-4.1.18-arm-9.patch

```

### 1d. Download Raspberry Pi specific patch  to kernel  (patch of Mathieu Rondonneau specific for hardware bcm-2709 (rpi2 and rpi3) )
```shell
wget  http://www.blaess.fr/christophe/files/article-2016-05-22/patch-xenomai-3-on-bcm-2709.patch

# mirror:

wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/patch-xenomai-3-on-bcm-2709.patch

```
#### patch applies on files :  (applies purely in kernel, doesn't effect drivers ): 
*  arch/arm/Kconfig
*  arch/arm/mach-bcm2709/armctrl.c
*  arch/arm/mach-bcm2709/bcm2708_gpio.c
*  arch/arm/mach-bcm2709/bcm2709.c
*  arch/arm/mach-bcm2709/include/mach/entry-macro.S
#
## 1e. Download a patched version of pinctrl-bcm2835.c from the official ipipe patch
```shell
wget -O pinctrl-bcm2835.c  http://git.xenomai.org/ipipe.git/plain/drivers/pinctrl/bcm/pinctrl-bcm2835.c?h=vendors/raspberry/ipipe-4.1

# mirror

wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/pinctrl-bcm2835.c

```
### 1f. Download a patch for script preapre_kernel.sh

```shell
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/prepare-kernel.patch
```

### 1g. Download a patch for Kconfig (drivers/xenomai/gpio/Kconfig)

```shell
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/Kconfig.patch
```
### 1h. Download a patch to support GCC6 to compile kernel source (drivers/char/broadcom/vc_sm/vmcs_sm.c)
```shell
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/vmcs_sm.patch
```


## 2.  Apply kernel patches ( xenomai+ipipe, bcm  )
---
### 2a. Apply xenomai (and ipipe) patch to kernel
**First check if the ipipe patch applies well: (because little bit different kernel)**
```shell
cd linux
patch  --dry-run  -p1  <  ../ipipe-core-4.1.18-arm-10.fixed.patch
```
 The argument --dry-run of the above execution allowed to verify if the patch is applied correctly without making changes.
 
 http://superuser.com/questions/755355/hunk-1-message-after-apply-a-patch

If the file to be patched has changed slightly since the patch was created, but the specific section remained the same, patch can detect that and apply the patch appropriately.
A hunk  message  means the file was successfully patched, but the first section that was patched was X lines earlier/later
than was originally specified.

#### Next to apply patch to prepare-kernel.sh script

this is needed for 'make deb-pkg' in the linux kernel source tree
      which otherwise makes packages with broken links!!

```shell
cd ../xenomai-3-3.0.5/scripts/
patch -p0  <  ../../prepare-kernel.patch
cd ../../
```

#### Now we can apply patch on kernel sources

```shell
xenomai-3-3.0.5/scripts/prepare-kernel.sh  --linux=linux/  --arch=arm  --ipipe=ipipe-core-4.1.18-arm-10.fixed.patch
```

#### At the end apply patch to Kconfig to add support gpio on BCM2709

this is needed for 'make deb-pkg' in the linux kernel source tree
      which otherwise makes packages with broken links!!

```shell
cd linux/drivers/xenomai/gpio/
patch -p0  <  ../../../../Kconfig.patch
cd ../../../../
```

### 2b. Apply Raspberry pi specific patch  to kernel  (patch of Mathieu Rondonneau specific for hardware bcm-2709 )
#
```shell
cd  linux/
# dry-run
patch  -p1  <  ../patch-xenomai-3-on-bcm-2709.patch  --dry-run
# if successfull  do real run
patch  -p1  <  ../patch-xenomai-3-on-bcm-2709.patch  
cd ..
```
### 2c. apply a patched version of pinctrl-bcm2835.c from the official ipipe patch
```shell
cp  -f pinctrl-bcm2835.c  linux/drivers/pinctrl/bcm/pinctrl-bcm2835.c 
```

### OPTIONAL: (use only if your compiler is GCC 6)  apply a patch to drivers/char/broadcom/vc_sm/vmcs_sm.c for support GCC6 compiler

```shell
cd linux/drivers/char/broadcom/vc_sm/
patch -p0  <  ../../../../../vmcs_sm.patch
cd ../../../../../
```

## 3. Configure linux kernel/module option
---
### Go to kernel source directory:
```shell
cd linux
```
### Create default config for bcm2709
```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- bcm2709_defconfig
```
 a default configuration for bcm2709 written to .config
### Specialize config for xenomai
 install package needed for menuconfig: 
 
```shell
sudo apt-get  install libncurses5-dev
```
### Run menuconfig
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- menuconfig

### Set the configurations according to the scheme:

    CPU Power Management  --->
      CPU Frequency scaling  --->
        [ ] CPU Frequency scaling
      CPU idle  --->
        [ ] CPU idle PM support
    Kernel Features  --->
        [ ] Contiguous Memory Allocator
        [ ] Allow for memory compaction
    Kernel Hacking  --->
        [ ] KGDB: kernel debugger
    Boot options  --->
        Kernel command line type --->
            [X] Extend bootloader kernel arguments


    Xenomai Cobalt --->
         Drivers  --->
                 Real-time GPIO drivers  --->
                    <M> GPIO controller                          => with " [*] GPIO controller  " instead it builds directly in kernel, then no need to load module anymore!
                    [*]   Support for BCM2835 GPIOs                 `-> however we stick to default and just build it as module and load it at boot in /etc/rc.local
                    [ ]   Enable GPIO core debugging features


    note: in above  [ ]  means disable,  [X]  means enable

### Return to main directory
```shell
cd ..
```

## 4. Build kernel and modules in linux directory 
--- 
### Go to kernel source directory:
```shell
cd linux
```
### Build kernel:
```shell
make -j $NUMCORES ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- zImage modules dtbs
```
### Install kernel in ../dist/ directoryt
```shell 
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- INSTALL_MOD_PATH=../dist modules_install
```
### Return to main directory
```shell
cd ..
```

## EXTRA: Install linux headers with Makefiles to build kernel modules on the pi itself
---
##   note: 
'make headers_install' installs only the headers, the Makefiles needed to build a kernel module are not there however with 'make deb-pkg' next to binary kernel packages it builds also a  linux-headers package which contains both headers and makefiles needed to build kernel modules. So we build this package and copy it to the pi, and install it as a debian package on the pi
```shell
cd linux/
export NUMCORES=4 
export KBUILD_DEBARCH=armhf 
make ARCH=arm -j $NUMCORES  CROSS_COMPILE=arm-linux-gnueabihf-  deb-pkg
```
### Copy the file  'linux-headers-*_armhf.deb' to the pi,then on the pi run :
```shell
   dpkg -i linux-headers-*_armhf.deb
```
it seems that the scripts in the linux headers itself are not cross-compiled to arm, (intel instead), so to use them to build kernel modules we first need to compile them as arm binaries, to
do this we can call 'make modules_prepare' which calls 'make scripts'
```shell
cd /usr/src/linux-headers-*/
# run following make command, which can give some errors, which should be ignored be giving it the -i option
make -i modules_prepare   
# building kernel modules should work fine now!  
cd ..
```


## 5. Built Xenomai user space part  ( kernel part is built with kernel above)
---

### You need to have installed autoconf and libtool :
```shell
sudo apt-get install autoconf
sudo apt-get install libtool
```

### Configure details see : https://xenomai.org/installing-xenomai-3-x/#_configuring
```shell
cd xenomai-3-3.0.5
./scripts/bootstrap 
./configure CFLAGS="-march=armv7-a  -mfloat-abi=hard -mfpu=neon -ffast-math" --host=arm-linux-gnueabihf --enable-smp --with-core=cobalt -enable-debug=partial

#
# note : http://xenomai.org/installing-xenomai-3-x/
#          –enable-smp
#              Turns on SMP support for Xenomai libraries.
#
#              Caution
#                  SMP support must be enabled in Xenomai libraries when the client
#                  applications are running over a SMP-capable kernel.

mkdir ../dist/xenomai
export DESTDIR=`realpath ../dist/xenomai`  # realpath because must be absolute path
make  DESTDIR="$DESTDIR"  install
```
## EXTRA: Compile Real-Time SPI driver for the Broadcom BCM2835 and BCM2836 SoCs using the RTDM API.
### Based on [https://github.com/nicolas-schurando/spi-bcm283x-rtdm](https://github.com/nicolas-schurando/spi-bcm283x-rtdm)
---
###1. Clone github repository
```shell
cd $BUILDDIR
git clone https://github.com/nicolas-schurando/spi-bcm283x-rtdm.git
```
### 2. Compile project using our kernel source
```shell
cd spi-bcm283x-rtdm
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- KERNEL_DIR=$BUILDDIR/linux/
```
Compiled kernel module should be on patch:
```shell
 $BUILDDIR/spi-bcm283x-rtdm/release/spi-bcm283x-rtdm.ko
```
Copy the generated kernel module onto the target and load it with the following command.
```shell
 sudo insmod spi-bcm283x-rtdm.ko
```
The system log should display something like:

```
[   59.577534] bcm283x_spi_rtdm_init: Starting driver ...
[   59.577582] bcm2835_init: Found device-tree node /soc.
[   59.577615] bcm2835_init: /soc/ranges = 0x7e000000 0x3f000000 0x01000000.
[   59.577638] bcm2835_init: Using device-tree values.
[   59.577998] mapmem: mapping 0x3f000000 for gpio succeded to 0xbe000000
[   59.578867] bcm283x_spi_rtdm_init: Device spidev0.0 registered without errors.
[   59.579362] bcm283x_spi_rtdm_init: Device spidev0.1 registered without errors.
```
## 6. Copy build stuff in dist/ to raspbian sdcard 
---
### 6.1. Mount sdcard
**Become root otherwise you cannot access sdcard**

```shell
sudo su -

export BUILDDIR=/tmp/rpi3
export SDCARD=/dev/sdX # X is name your device
export MOUNTPOINT=/media/sdcard

mkdir ${MOUNTPOINT}
mount ${SDCARD}2 ${MOUNTPOINT}
mount ${SDCARD}1 ${MOUNTPOINT}/boot

cd $BUILDDIR
```
### 6.2. Copy Xenomai user space files to sd card

```shell
cd dist/xenomai
chown -R root:root *
cp -a * ${MOUNTPOINT}/
cd ../..
```

### 6.3. Copy kernel and modules 

# 6.3a. Copy kernel and device tree files from linux dir  to /boot/ directory on image
```shell
cd linux/
cp arch/arm/boot/zImage ${MOUNTPOINT}/boot/
cp arch/arm/boot/dts/bcm27*.dtb ${MOUNTPOINT}/boot/
rm -rf ${MOUNTPOINT}/boot/overlays/*
cp arch/arm/boot/dts/overlays/*.dtb* ${MOUNTPOINT}/boot/overlays/
```
### 6.3b. Copy modules from dist/lib/modules to sd card
```shell
cd ../
cp -r dist/lib/modules/* ${MOUNTPOINT}/lib/modules
```

#### Update config for new kernel and device tree at end of /boot/config.txt add :
```shell
nano ${MOUNTPOINT}/boot/config.txt
```

```shell
    kernel=zImage
    # for pi2
    #device_tree=bcm2709-rpi-2-b.dtb
    # for pi3
    device_tree=bcm2710-rpi-3-b.dtb
```

#### Update: if you just rename the just installed file '/boot/zImage' to '/boot/kernel7'  then you don't need to edit /boot/config.txt because the bootscript on the raspberry pi finds out it is running on pi2 or pi3 and the automatically boots '/boot/kernel7'.   
####Note: on pi0 or pi1 it boots '/boot/kernel' instead!

### At the end:

```shell
umount ${MOUNTPOINT}/boot ${MOUNTPOINT}/
```
---
## credits :

Documentation how to build kernel for raspbian:
    [https://www.raspberrypi.org/documentation/linux/kernel/building.md](https://www.raspberrypi.org/documentation/linux/kernel/building.md)

### document based on:
*  [http://www.cs.kun.nl/lab/xenomai/](http://www.cs.kun.nl/lab/xenomai/)
*  [https://github.com/harcokuppens/xenomai3_rpi_gpio](https://github.com/harcokuppens/xenomai3_rpi_gpio)
*  [http://www.blaess.fr/christophe/2016/05/22/xenomai-3-sur-raspberry-pi-2/](http://www.blaess.fr/christophe/2016/05/22/xenomai-3-sur-raspberry-pi-2/)
*  [http://wiki.csie.ncku.edu.tw/embedded/xenomai#xenomai-3-on-raspberry-pi-3](http://wiki.csie.ncku.edu.tw/embedded/xenomai#xenomai-3-on-raspberry-pi-3)
*  [https://xenomai.org/installing-xenomai-3-x/](https://xenomai.org/installing-xenomai-3-x/)
*  [https://github.com/nicolas-schurando/spi-bcm283x-rtdm](https://github.com/nicolas-schurando/spi-bcm283x-rtdm)

