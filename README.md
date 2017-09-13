# [HOW TO] Built Raspbian Linux kernel with xenomai 3.0.5 path for Rasberry Pi 3 on Ubuntu


## 0. Prepare environment
---
```shell
# install git and subversion
sudo apt-get update
sudo apt-get install git subversion

# Preapre variables
export BUILDDIR=/var/tmp/rpi2
# build kernel with NUMCORES
export NUMCORES=4

# directory where download and build everything  
mkdir $BUILDDIR   
#  directory for ready to install files
mkdir  $BUILDDIR/dist  
```

## 1. Download sources 
---
####Go to $BUILDDIR directory
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
####alternative and faster version:
```shell
svn export https://github.com/SPFCG/xenomai_kernel/trunk/mirror/rpi-4.1.y/linux linux
```

### 1b. Download Xenomai 3.0.5 source
```shell
wget http://git.xenomai.org/xenomai-3.git/snapshot/xenomai-3-3.0.5.tar.bz2
tar -xjvf xenomai-3-3.0.5.tar.bz2
```
####alternative version:
```shell
svn export https://github.com/SPFCG/xenomai_kernel/trunk/mirror/xenomai-3-3.0.5 xenomai-3-3.0.5
```

### 1c. Download ipipe  patch  (effects both kernel and some drivers(using interrupts))
```shell
wget https://xenomai.org/downloads/ipipe/v4.x/arm/ipipe-core-4.1.18-arm-10.patch

#older: https://xenomai.org/downloads/ipipe/v4.x/arm/older/ipipe-core-4.1.18-arm-9.patch

```
### Download patched ipipe with small trivial fix for kernel configuration on bcm-2709 

```shell
cd ../xenomai-3-3.0.5/scripts/
```
####original version:
```shell
wget https://xenomai.org/downloads/ipipe/v4.x/arm/ipipe-core-4.1.18-arm-10.patch
#older:
wget https://xenomai.org/downloads/ipipe/v4.x/arm/older/ipipe-core-4.1.18-arm-9.patch


#mirrors:

wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/ipipe-core-4.1.18-arm-10.patch
#older
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/ipipe-core-4.1.18-arm-9.patch

```

### 1d. Download Raspberry Pi specific patch  to kernel  (patch of Mathieu Rondonneau specific for hardware bcm-2709 (rpi2 and rpi3) )
```shell
wget  http://www.blaess.fr/christophe/files/article-2016-05-22/patch-xenomai-3-on-bcm-2709.patch

#mirror:

wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/patch-xenomai-3-on-bcm-2709.patch

```
#### patch applies on files :  (applies purely in kernel, doesn't effect drivers )
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
## 1f. Download a patch for script preapre_kernel.sh

```shell
wget https://raw.githubusercontent.com/SPFCG/xenomai_kernel/master/patches/prepare-kernel.patch
```

## 2.  Apply kernel patches ( xenomai+ipipe, bcm  )
---
### 2a. Apply xenomai (and ipipe) patch to kernel
# 
#### first check if the ipipe patch applies well: (because little bit different kernel)
```shell
cd linux
patch  --dry-run  -p1  <  ../ipipe-core-4.1.18-arm-10.fixed.patch
```
 The argument --dry-run of the above execution allowed to verify if the patch is applied correctly without making changes.
 
 http://superuser.com/questions/755355/hunk-1-message-after-apply-a-patch

If the file to be patched has changed slightly since the patch was created, but the specific section remained the same, patch can detect that and apply the patch appropriately.
A hunk  message  means the file was successfully patched, but the first section that was patched was X lines earlier/later
than was originally specified.

#### next to apply patch to prepare-kernel.sh script

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

## 3. Configure linux kernel/module option
---
#### Go to kernel source directory:
```shell

cd linux
```