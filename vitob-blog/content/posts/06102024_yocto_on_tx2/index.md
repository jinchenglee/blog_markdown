+++
title = 'Yocto on TX2 with Jetpack4.3'
date = 2024-06-10T17:08:34-07:00
draft = false
toc = true
+++


#Yocto on TX2 with Jetpack4.3

This blog tries to build TX2 image with industry-default flow for embedded system -- the Yocto project, for ancient Jetson TX2 with Jetpack 4.3 (L4T R32.3.1). 

## Default meta-tegra build

* Sync down meta-tegra with right branch [`dunfell-l4t-r32.3.1`](https://github.com/OE4T/meta-tegra/tree/dunfell-l4t-r32.3.1). Also, please check [L4T R32.3.1 Notes](https://github.com/OE4T/meta-tegra/wiki/L4T-R32.3.1-Notes).
```
cd ~ && mkdir yocto-tx2-jp43 && cd yocto-tx2-jp43
git clone https://github.com/OE4T/meta-tegra.git 
cd meta-tegra
git branch -a
git checkout dunfell-l4t-r32.3.1
```

* Sync down poky with right branch `dunfell`. 
```
cd ~/yocto-tx2-jp32
git clone https://github.com/yoctoproject/poky.git
cd poky
git branch -a
git checkout dunfell
```

* Active environment
```
source poky/oe-init-build-env
```

* Add configurations to build/conf/local.conf
```
MACHINE ??= "jetson-tx2"

DL_DIR ?= "/home/${USER}/Yocto/downloads"

SSTATE_DIR ?= "/home/${USER}/Yocto/sstate_dir"

IMAGE_CLASSES += "image_types_tegra"

PREFERRED_VERSION_python3 = "3.8%"
PREFERRED_VERSION_python3-native = "3.8%"

BB_NUMBER_THREADS = '11'
PARALLEL_MAKE = '-j11'
```

* Add configuration to build/conf/bblayers.conf
```
/home/${USER}/yocto-tegra/meta-tegra \
```
The bblayers.conf looks like:
```
#// POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
#// changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/${USER}/yocto-tegra/meta-tegra \
  /home/${USER}/yocto-tx2-jp43/poky/meta \
  /home/${USER}/yocto-tx2-jp43/poky/meta-poky \
  /home/${USER}/yocto-tx2-jp43/poky/meta-yocto-bsp \
  "
```

### Build core-image-minimal.
```
bitbake core-image-minimal
```

Successful completion of the built shows sth. like below. I'm running on a previously built cache, so a lot of tasks don't need re-run. If you are running from scratch, this process can take pretty long depending on your host machine (~hours). 

```
@ubuntu18:~/yocto-tx2-jp43/build$ bitbake core-image-minimal
Loading cache: 100% |#####################################################################################################| Time: 0:00:00
Loaded 1486 entries from dependency cache.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "1.46.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "ubuntu-18.04"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "jetson-tx2"
DISTRO               = "poky"
DISTRO_VERSION       = "3.1.33"
TUNE_FEATURES        = "aarch64 armv8a crc"
TARGET_FPU           = ""
meta-tegra           = "dunfell-l4t-r32.3.1:12ca32302b4d23045d1dc19102d0c21670052f7d"
meta                 
meta-poky            
meta-yocto-bsp       = "dunfell:63d05fc061006bf1a88630d6d91cdc76ea33fbf2"

Initialising tasks: 100% |################################################################################################| Time: 0:00:01
Sstate summary: Wanted 1203 Found 1199 Missed 4 Current 0 (99% match, 0% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3200 tasks of which 2952 didn't need to be rerun and all succeeded.
```

* Deploy

Using this script:
```
#!/bin/bash

image=$1
machine=$2

scriptdir="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null && pwd )"
deployfile=${image}-${machine}.tegraflash.zip
tmpdir=`mktemp`

rm -rf $tmpdir
mkdir -p $tmpdir
echo "Using temp directory $tmpdir"
pushd $tmpdir
cp $scriptdir/build/tmp/deploy/images/${machine}/$deployfile .
unzip $deployfile
set -e
sudo ./doflash.sh
popd
echo "Removing temp directory $tmpdir"
rm -rf $tmpdir
```

Put Jetson TX2 development kit into force recovery mode by:
1. Press down REC button and not release
2. Press down POWER button and release
3. Release REC button

To make sure the board has entered recoverage mode, check lsusb on host:
```
xxx@ubuntu18:~/yocto-tegra$ lsusb
...
Bus 003 Device 013: ID 0955:7c18 NVidia Corp.                  <= Check this line. 
...
```

Run the deploy command:
```
:~/yocto-tx2-jp43$ sudo ./deploy.sh core-image-minimal jetson-tx2
```

On the debug serial port or the connected monitor to TX2, you should see this:
```
Poky (Yocto Project Reference Distro) 3.1.33 jetson-tx2 /dev/ttyXXX

jetson-tx2 login:
```

Using `root` without password should login you in. 

### Build core-image-sato-dev

Similarly, we can build an image with GUI from yocto:
```
bitbake core-image-sato-dev
```

Using the same deploy.sh script to flash:
```
sudo ./deploy.sh core-image-sato-dev jetson-tx2
```


## Patch kernel with PREEMPT-RT

PREEMPT-RT patches are for real-time applications on Linux. The patching process described below refers to these links:
1. [PREEMPT-RT patches for Jetson Nano](https://forums.developer.nvidia.com/t/preempt-rt-patches-for-jetson-nano/72941)
2. [Applying PREEMPT RT Patches dunfell l4t r32.4.3](https://github.com/OE4T/meta-tegra/wiki/Applying-PREEMPT-RT-Patches-dunfell-l4t-r32.4.3)

### Getting Ready for kernel-dev Using devtool according to [Yocto Doc](9https://docs.yoctoproject.org/kernel-dev/common.html#getting-ready-to-develop-using-devtool).

Update local.conf file:
```
MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "kernel-modules"
```

* Create a layer for patches and inform BitBake build env about the newly added layer:
```
xxx@ubuntu18:~/yocto-tx2-jp43$ cd build/
xxx@ubuntu18:~/yocto-tx2-jp43/build$ bitbake-layers create-layer ../meta-tx2-jp43
NOTE: Starting bitbake server...
Add your new layer with 'bitbake-layers add-layer ../meta-tx2-jp43'

xxx@ubuntu18:~/yocto-tx2-jp43/build$ bitbake-layers add-layer ../meta-tx2-jp43/
NOTE: Starting bitbake server...
```
The last commad will add the newly added layer to your build/conf/bblayers.conf
```
xxx@ubuntu18:~/yocto-tx2-jp43/build$ cat conf/bblayers.conf 
#// POKY_BBLAYERS_CONF_VERSION is increased each time build/conf/bblayers.conf
#// changes incompatibly
POKY_BBLAYERS_CONF_VERSION = "2"

BBPATH = "${TOPDIR}"
BBFILES ?= ""

BBLAYERS ?= " \
  /home/${USER}/yocto-tegra/meta-tegra \
  /home/${USER}/yocto-tx2-jp43/poky/meta \
  /home/${USER}/yocto-tx2-jp43/poky/meta-poky \
  /home/${USER}/yocto-tx2-jp43/poky/meta-yocto-bsp \
  /home/<your user name>/yocto-tx2-jp43/meta-tx2-jp43 \
  "
```
* Bitbake core-image-minimal to make sure everything is ok.
```
@ubuntu18:~/yocto-tx2-jp43/build$ bitbake core-image-minimal
Parsing recipes: 100% |###################################################################################################| Time: 0:00:12
Parsing of 876 .bb files complete (0 cached, 876 parsed). 1487 targets, 74 skipped, 0 masked, 0 errors.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "1.46.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "jetson-tx2"
DISTRO               = "poky"
DISTRO_VERSION       = "3.1.33"
TUNE_FEATURES        = "aarch64 armv8a crc"
TARGET_FPU           = ""
meta-tegra           = "dunfell-l4t-r32.3.1:12ca32302b4d23045d1dc19102d0c21670052f7d"
meta                 
meta-poky            
meta-yocto-bsp       = "dunfell:63d05fc061006bf1a88630d6d91cdc76ea33fbf2"
meta-tx2-jp43        = "<unknown>:<unknown>"         <= NOTICE the blank "UNKNOWN" here!

Initialising tasks: 100% |################################################################################################| Time: 0:00:01
Sstate summary: Wanted 367 Found 359 Missed 8 Current 836 (97% match, 99% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 3200 tasks of which 3186 didn't need to be rerun and all succeeded.
```

### Using devtool to Patch the Kernel

According to [Yocto Doc](https://docs.yoctoproject.org/kernel-dev/common.html#using-devtool-to-patch-the-kernel).

* Check out kernel source files.
```
xxx@ubuntu18:~/yocto-tx2-jp43/build$ devtool modify linux-tegra
NOTE: Starting bitbake server...
NOTE: Reconnecting to bitbake server...
NOTE: Retrying server connection (#1)...
Loading cache: 100% |#####################################################################################################| Time: 0:00:00
Loaded 1487 entries from dependency cache.
NOTE: Resolving any missing task queue dependencies

Build Configuration:
BB_VERSION           = "1.46.0"
BUILD_SYS            = "x86_64-linux"
NATIVELSBSTRING      = "universal"
TARGET_SYS           = "aarch64-poky-linux"
MACHINE              = "jetson-tx2"
DISTRO               = "poky"
DISTRO_VERSION       = "3.1.33"
TUNE_FEATURES        = "aarch64 armv8a crc"
TARGET_FPU           = ""
meta-tegra           = "dunfell-l4t-r32.3.1:12ca32302b4d23045d1dc19102d0c21670052f7d"
meta                 
meta-poky            
meta-yocto-bsp       = "dunfell:63d05fc061006bf1a88630d6d91cdc76ea33fbf2"
meta-tx2-jp43        
workspace            = "<unknown>:<unknown>"

Initialising tasks: 100% |################################################################################################| Time: 0:00:00
Sstate summary: Wanted 55 Found 54 Missed 1 Current 50 (98% match, 99% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 474 tasks of which 463 didn't need to be rerun and all succeeded.
INFO: Adding local source files to srctree...
INFO: Copying kernel config to srctree
INFO: Source tree extracted to /home/xxx/yocto-tx2-jp43/build/workspace/sources/linux-tegra
INFO: Recipe linux-tegra now set up to build from /home/xxx/yocto-tx2-jp43/build/workspace/sources/linux-tegra
```

* Patch the kernel source and build the kernel:

```
xxx@ubuntu18:~/yocto-tx2-jp43/build/workspace/sources/linux-tegra$ ./scripts/rt-patch.sh apply-patches
PREEMPT RT patches successfully applied for Auto!
PREEMPT RT patches successfully applied for L4T!
```
Using git status can see a bunch of files are modified and another bunch are added. However, when trying to build kernel exposes build errors:
```
xxx@ubuntu18:~/yocto-tx2-jp43/build$ devtool build linux-tegra
NOTE: Starting bitbake server...
NOTE: Reconnecting to bitbake server...
NOTE: Retrying server connection (#1)...
...
NOTE: Executing Tasks
NOTE: linux-tegra: compiling from external source tree /home/xxx/yocto-tx2-jp43/build/workspace/sources/linux-tegra
ERROR: linux-tegra-4.9.140+git999-r0 do_compile_kernelmodules: oe_runmake failed
ERROR: linux-tegra-4.9.140+git999-r0 do_compile_kernelmodules: Execution of '/home/xxx/yocto-tx2-jp43/build/tmp/work/jetson_tx2-poky-linux/linux-tegra/4.9.140+git999-r0/temp/run.do_compile_kernelmodules.17505' failed with exit code 1
ERROR: Logfile of failure stored in: /home/xxx/yocto-tx2-jp43/build/tmp/work/jetson_tx2-poky-linux/linux-tegra/4.9.140+git999-r0/temp/log.do_compile_kernelmodules.17505

(Details:)
...
/home/xxx/yocto-tx2-jp43/build/workspace/sources/linux-tegra/nvidia/drivers/net/wireless/bcmdhd/dhd_pno.c:1606:23: error: passing argument 1 of 'waitqueue_active' from incompatible pointer type [-Werror=incompatible-pointer-types]
 1606 |  if (waitqueue_active(&_pno_state->get_batch_done.wait))
      |                       ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
      |                       |
      |                       struct swait_queue_head *
...

$ vim ~/yocto-tx2-jp43/build/workspace/sources/linux-tegra/nvidia/drivers/net/wireless/bcmdhd/dhd_pno.c
1600 #if IS_ENABLED(CONFIG_PREEMPT_RT_FULL)
1601         if (swait_active(&_pno_state->get_batch_done.wait))
1602 #else
1603 #if LINUX_VERSION_CODE >= KERNEL_VERSION(4, 14, 57)
1604         if (waitqueue_active((struct wait_queue_head *)&_pno_state->get_batch_done.wait))
1605 #else
1606         if (waitqueue_active(&_pno_state->get_batch_done.wait))     <= ERROR HERE!
1607 #endif
1608 #endif
```

* Modify kernel configuration

Google around shows the error could be avoided if `CONFIG_PREEMPT_RT_FULL` is defined in .config before kernel compilation. (BTW, there is another fix option [here](https://github.com/rockchip-linux/kernel/issues/261#issuecomment-1003837100) but I don't try this way.)
```
xxx@ubuntu18:~/yocto-tx2-jp43/build$ bitbake -c menuconfig virtual/kernel
```
Select "Kernel features -> Preemption Model (...)" and select "Fully Preemptible Kernel (RT)". 

* Save the modified kernel configuration
```
$ bitbake -c savedefconfig virtual/kernel
$ cp ./tmp/work/jetson_tx2-poky-linux/linux-tegra/4.9.140+git999-r0/linux-tegra-4.9.140+git999/defconfig ./workspace/sources/linux-tegra/arch/arm64/configs/defconfig
```

Now compile again, it should finish successfully.
```
xxx@ubuntu18:~/yocto-tx2-jp43/build$ devtool build linux-tegra
NOTE: Starting bitbake server...
NOTE: Reconnecting to bitbake server...
NOTE: Retrying server connection (#1)...
...
WARNING: /home/xxx/yocto-tegra/meta-tegra/recipes-kernel/linux/linux-tegra_4.9.bb:do_compile is tainted from a forced run ETA:  0:00:00
Initialising tasks: 100% |################################################################################################| Time: 0:00:02
Sstate summary: Wanted 324 Found 322 Missed 2 Current 698 (99% match, 99% complete)
NOTE: Executing Tasks
NOTE: linux-tegra: compiling from external source tree /home/xxx/yocto-tx2-jp43/build/workspace/sources/linux-tegra
NOTE: Tasks Summary: Attempted 2648 tasks of which 2627 didn't need to be rerun and all succeeded.

Summary: There was 1 WARNING message shown.
```

* Create the image with modified kernel
```
$ devtool build-image core-image-sato-dev
```
Flash the TX2 and you can see the patched kernel bootstrap successfully.
```
root@jetson-tx2:~# uname -a
Linux jetson-tx2 4.9.140-rt93-l4t-r32.3.1+ga0004d2ad6a4 #1 SMP PREEMPT RT Mon Jun 10 18:20:28 UTC 2024 aarch64 aarch64 aarch64 GNU/Linux
```


### Save the patch work for future use

* Stage and commit local changes to the kernel
```
$ cd workspace/sources/linux-tegra

-- Commit everything (new and modifed) ---
commit 9d4a27452b73870aa07a13883818284c310331e3 (HEAD -> patches-l4t-r32.3.1)
Author: OpenEmbedded <oe.patch@oe>
Date:   Mon Jun 10 12:38:42 2024 -0700

    My patch for Jetson TX2 with Jetpack 4.3/L4T R32.3.1 and PREEMPT-RT.

-- Save a copy of defconfig and localversion_auto.cfg to DL_DIR (defined in local.conf).
$ cp defconfig ~/Yocto/downloads/
$ cp localversion_auto.cfg ~/Yocto/downloads/

-- Export the changes in the commit as patches and create a .bbappend file in layer specificed (meta-tx2-jp43): 
$ devtool finish linux-tegra ../meta-tx2-jp43

$:~/yocto-tx2-jp43$ tree meta-tx2-jp43
meta-tx2-jp43
├── conf
│   └── layer.conf
├── COPYING.MIT
├── README
...
└── recipes-kernel
    └── linux
        ├── linux-tegra-4.9
        │   ├── 0001-My-patch-for-Jetson-TX2-with-Jetpack-4.3-L4T-R32.3.1.patch
        │   └── devtool-fragment.cfg
        └── linux-tegra_4.9.bb
```

* Create a repository for the layer
Created the layer on github [repo](https://github.com/jinchenglee/meta-tx2-jp43). 


## Yocto build with RT patch from scratch

Now we can use Yocto and our customized layer to build RT-patched Linux from scratch for Jetson TX2 with Jetpack 4.3 and L4T R32.3.1.

```
git clone https://github.com/OE4T/meta-tegra.git 
cd meta-tegra && git checkout dunfell-l4t-r32.3.1 && cd ..

git clone https://github.com/yoctoproject/poky.git
cd poky && git checkout dunfell && cd ..

git clone https://github.com/jinchenglee/meta-tx2-jp43

source poky/oe-init-build-env

#// Make necessary changes to build/conf/local.conf and build/conf/bblayers.conf
#// For example, add below to local.conf:
#//
#//EXTRA_IMAGE_FEATURES ?= "debug-tweaks tools-sdk package-management"
#//PACKAGE_CLASSES ?= "package_deb"


bitbake core-image-sato-dev
```
