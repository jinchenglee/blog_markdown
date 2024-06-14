+++
title = 'Yocto kernel patch L4T R32.3.1 on tx2'
date = 2024-06-13T21:42:50-07:00
draft = true
toc = true
+++

## Patch to fix `*-sdk` build failures

When a unmodified yocto build environment is setup (see previous blog about Yocto on TX2, "default meta-tegra build"), trying to build `*-sdk` (e.g. `core-image-sato-sdk`) will fail although `*-dev` builds are fine. 


### compilation: srcline.c

The failure is related to `perf-1.0` tool that Linux kernel requires. This first error comes from kernel compilation of `tools/perf/util/srcline.c`. 

```
/build/tmp/work/intel_core2_32-poky-linux/perf/1.0-r9/perf-1.0/perf-in.o: in function `find_address_in_section':
/usr/src/debug/perf/1.0-r9/perf-1.0/tools/perf/util/srcline.c:200: undefined reference to `bfd_get_section_flags'
```

A search online [result](https://lists.yoctoproject.org/g/meta-intel/topic/patch_2_2_linux_intel/71501209) shows hint how to fix the source code. 


### installation: setup.py

Then the failure comes from (it seems) `PREFERRED_VERSION_python3 = "3.8%"` vs. `tools/perf/util/setup.py` of `perf-1.0` expects python2.x. After searching around, I ended up using [2to3 - Automated Python 2 to 3 code translation](https://docs.python.org/3/library/2to3.html) to have converted the setup.py which resulted the error. 


### installation: python.c 

The installation phase of `perf-1.0` hit yet another failure, which seems trying to expose the built perf library to python, but again, it expects python 2.x. The error is related to Python C-API changes on PyObject_HEAD macro used in `tools/perf/util/python.c`. Several lines need to be changes similarly as below:

```
@@ -513,7 +513,9 @@ static int pyrf_cpu_map__init(struct pyrf_cpu_map *pcpus,
 static void pyrf_cpu_map__delete(struct pyrf_cpu_map *pcpus)
 {
        cpu_map__put(pcpus->cpus);
-       pcpus->ob_type->tp_free((PyObject*)pcpus);
+       // Python2 vs. 3 incompatibility
+       //pcpus->ob_type->tp_free((PyObject*)pcpus);
+       Py_TYPE(pcpus)->tp_free((PyObject*)pcpus);
 }

```


### Using devtool to Patch the Kernel

Yocto provided easty-to-follow [procedure](https://docs.yoctoproject.org/kernel-dev/common.html#using-devtool-to-patch-the-kernel) to patch the kernel by using the devtool tool. It will fetch/sync the specifric kernel version to a local workspace. After local modification/try build/verification, it will package the changes as a recipe automatically. Really convenient. 

Notice: unlike what mentioned in the yocto document `linux-yocto`, for our target tx2, we should replace that with `linux-tegra`. 

The auto-prepared recipe looks like this:
```
./meta-example/
├── conf
│   └── layer.conf
├── COPYING.MIT
├── README
└── recipes-kernel
    └── linux
        ├── linux-tegra
        │   └── 0001-Workaround-build-installation-issues-related-to-perf.patch
        └── linux-tegra_4.9.bbappend
```

### Rebuild and flash

Rebuild the image shows suceess.

```
@ubuntu18:~/yocto-tegra/build$ bitbake core-image-sato-sdk 
Parsing recipes: 100% |#################################################################################################| Time: 0:00:12
Parsing of 875 .bb files complete (0 cached, 875 parsed). 1486 targets, 76 skipped, 0 masked, 0 errors.
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
meta-example        = "<unknown>:<unknown>"

Initialising tasks: 100% |##############################################################################################| Time: 0:00:04
Sstate summary: Wanted 3202 Found 3158 Missed 44 Current 0 (98% match, 0% complete)
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 7674 tasks of which 7013 didn't need to be rerun and all succeeded.
```

Flash with command `sudo ./deploy.sh core-image-sato-sdk  jetson-tx2`.


## Post-flash: Apt-get sources.list and Cyclictest

Some post-flash actions on the TX2 device to prepare some comparison vs. a later PREEMPT-RT patched kernel. Will need `rt-tests` tool cyclictest on the device. 

* Set up apt-get sources.list

Our built image comes with apt-get because we've set added these to build/conf/local.conf (maybe only the package_deb is necessary for that):

```
MACHINE = "jetson-tx2"
MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "kernel-modules"

DL_DIR ?= "/home/${USER}/Yocto/downloads"
SSTATE_DIR ?= "/home/${USER}/Yocto/sstate_dir"

PACKAGE_CLASSES ?= "package_deb"

EXTRA_IMAGE_FEATURES ?= "debug-tweaks tools-sdk package-management"
IMAGE_CLASSES += "image_types_tegra"

PREFERRED_VERSION_python3 = "3.8%"
PREFERRED_VERSION_python3-native = "3.8%"

BB_NUMBER_THREADS = '11'
PARALLEL_MAKE = '-j11'
```

After tx2 bootstrapped, set this to [/etc/apt/sources.list](https://gist.github.com/josephlr/5034c933bbcfddc25a9275037821b048):

```
deb [arch=amd64,i386] http://us.archive.ubuntu.com/ubuntu/ bionic main restricted universe multiverse
deb [arch=amd64,i386] http://us.archive.ubuntu.com/ubuntu/ bionic-updates main restricted universe multiverse
deb [arch=amd64,i386] http://us.archive.ubuntu.com/ubuntu/ bionic-backports main restricted universe multiverse
deb [arch=amd64,i386] http://security.ubuntu.com/ubuntu bionic-security main restricted universe multiverse

deb [arch=arm64,armhf,ppc64el,s390x] http://ports.ubuntu.com/ubuntu-ports/ bionic main restricted universe multiverse
deb [arch=arm64,armhf,ppc64el,s390x] http://ports.ubuntu.com/ubuntu-ports/ bionic-updates main restricted universe multiverse
deb [arch=arm64,armhf,ppc64el,s390x] http://ports.ubuntu.com/ubuntu-ports/ bionic-backports main restricted universe multiverse
deb [arch=arm64,armhf,ppc64el,s390x] http://ports.ubuntu.com/ubuntu-ports/ bionic-security main restricted universe multiverse
```

* Install rt-tests

Then do `apt-get update`, then install rt-tests:

```
apt-get install rt-tests
```

It complains about below, which might be resolved by `dpkg -i --force-overwrite <filename>.deb` to force overwrite, but I don't bother to do that, since , but it seems `cyclictest` has been installed successfully. 

```
Preparing to unpack .../xz-utils_5.2.2-1.3ubuntu0.1_arm64.deb ...
Unpacking xz-utils (5.2.2-1.3ubuntu0.1) ...
dpkg: error processing archive /var/cache/apt/archives/xz-utils_5.2.2-1.3ubuntu0.1_arm64.deb (--unpack):
 trying to overwrite '/usr/bin/lzmainfo', which is also in package xz 5.2.4-r0
dpkg-deb: error: paste subprocess was killed by signal (Broken pipe)
Selecting previously unselected package rt-tests.
Preparing to unpack .../rt-tests_1.0-3_arm64.deb ...
Unpacking rt-tests (1.0-3) ...
Errors were encountered while processing:
 /var/cache/apt/archives/xz-utils_5.2.2-1.3ubuntu0.1_arm64.deb
E: Sub-process /usr/bin/dpkg returned an error code (1)
```

* run cyclictest

```
root@jetson-tx2:~# uname -a
Linux jetson-tx2 4.9.140-l4t-r32.3.1+ga0004d2ad6a4 #1 SMP PREEMPT ... aarch64 aarch64 aarch64 GNU/Linux

root@jetson-tx2:~#  cyclictest -t 5 -p 80 -n
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.06 0.08 0.08 1/222 3921          

T: 0 ( 3913) P:80 I:1000 C: 297462 Min:      7 Act:   29 Avg:   31 Max:     334
T: 1 ( 3914) P:80 I:1500 C: 198308 Min:      6 Act:   28 Avg:   29 Max:     230
T: 2 ( 3915) P:80 I:2000 C: 148731 Min:      8 Act:   23 Avg:   31 Max:     288
T: 3 ( 3916) P:80 I:2500 C: 118985 Min:      6 Act:   43 Avg:   29 Max:     234
T: 4 ( 3917) P:80 I:3000 C:  99154 Min:      6 Act:   23 Avg:   30 Max:     201
```


## Patch PREEMPT-RT

These links are referred to as doing the patch: 

* meta-tegra: [Applying PREEMPT RT Patches dunfell l4t r32.4.3](Applying PREEMPT RT Patches dunfell l4t r32.4.3)
* NVidia forum: [PREEMPT-RT patches for Jetson Nano](https://forums.developer.nvidia.com/t/preempt-rt-patches-for-jetson-nano/72941)

Recorded steps:

```
pwd: ~/yocto-tx2-jp43-rt/build
bitbake-layers create-layer ../meta-tx2-jp43
bitbake-layers add-layer ../meta-tx2-jp43
devtool modify linux-tegra

pwd: ~/yocto-tx2-jp43-rt/build/workspace/sources/linux-tegra/scripts
./rt-patch.sh apply-patches

pwd: ~/yocto-tx2-jp43-rt/build
devtool menuconfig linux-tegra
```

Configure these:
```
General setup -> Timer subsystem -> Timer tick handling -> Full dynticks system (tickless)
Kernel Features -> Preemption  Model: Fully Preemptible Kernel (RT)
Optional: Kernel Features -> Timer frequency: 1000 HZ (default is 250Hz)
```

The changes in menuconfig is logged in file `~/yocto-tx2-jp43-rt/build/workspace/sources/linux-tegra/oe-local-files/devtool-fragment.cfg`:

```
# CONFIG_NO_HZ_IDLE is not set
CONFIG_NO_HZ_FULL=y
# CONFIG_NO_HZ_FULL_ALL is not set
# CONFIG_NO_HZ_FULL_SYSIDLE is not set
CONFIG_VIRT_CPU_ACCOUNTING=y
# CONFIG_TICK_CPU_ACCOUNTING is not set
CONFIG_VIRT_CPU_ACCOUNTING_GEN=y
CONFIG_PREEMPT_RCU=y
CONFIG_CONTEXT_TRACKING=y
# CONFIG_CONTEXT_TRACKING_FORCE is not set
CONFIG_RCU_NOCB_CPU=y
CONFIG_RCU_NOCB_CPU_NONE=y
# CONFIG_RCU_NOCB_CPU_ZERO is not set
# CONFIG_RCU_NOCB_CPU_ALL is not set
CONFIG_PREEMPT=y
CONFIG_PREEMPT_RT_BASE=y
CONFIG_PREEMPT_LAZY=y
# CONFIG_PREEMPT_NONE is not set
CONFIG_PREEMPT_RT_FULL=y
CONFIG_PREEMPT_COUNT=y
# CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS is not set
CONFIG_DEBUG_PREEMPT=y
# CONFIG_PREEMPT_TRACER is not set
```

Continue building kernel and image:

```
devtool build linux-tegra
devtool build-image core-image-sato-dev
```

## Combine both patches

```
@ubuntu18:~/yocto-tx2-jp43-rt$ tree ./meta-tx2-jp43/
./meta-tx2-jp43/
├── conf
│   └── layer.conf
├── COPYING.MIT
├── README
└── recipes-kernel
    └── linux
        ├── linux-tegra
        │   ├── 0001-Apply-PREEMPT-RT-patch-to-patches-l4t-r32.3.1-on-lin.patch
        │   ├── 0002-Workaround-build-installation-issues-related-to-perf.patch
        │   └── devtool-fragment.cfg
        └── linux-tegra_%.bbappend

@ubuntu18:~/yocto-tx2-jp43-rt$ cat meta-tx2-jp43/recipes-kernel/linux/linux-tegra_%.bbappend 
FILESEXTRAPATHS_prepend := "${THISDIR}/${PN}:"

SRC_URI += "file://devtool-fragment.cfg file://0001-Apply-PREEMPT-RT-patch-to-patches-l4t-r32.3.1-on-lin.patch"
SRC_URI += "file://0002-Workaround-build-installation-issues-related-to-perf.patch"

@ubuntu18:~/yocto-tx2-jp43-rt/build$ bitbake core-image-sato-sdk
```

Run cyclictest (can compare with the run w/o RT patch):
```
root@jetson-tx2:~# uname -a
Linux jetson-tx2 4.9.140-rt93-l4t-r32.3.1+ga0004d2ad6a4 #1 SMP PREEMPT RT ... aarch64 aarch64 aarch64 GNU/Linux

root@jetson-tx2:~#  cyclictest -t 5 -p 80 -n
# /dev/cpu_dma_latency set to 0us
policy: fifo: loadavg: 0.17 0.25 0.18 2/306 3979          

T: 0 ( 3971) P:80 I:1000 C: 279669 Min:      7 Act:   31 Avg:   29 Max:     127
T: 1 ( 3972) P:80 I:1500 C: 186446 Min:      7 Act:   29 Avg:   30 Max:     139
T: 2 ( 3973) P:80 I:2000 C: 139834 Min:      8 Act:   24 Avg:   31 Max:     146
T: 3 ( 3974) P:80 I:2500 C: 111867 Min:      8 Act:   29 Avg:   31 Max:     117
T: 4 ( 3975) P:80 I:3000 C:  93223 Min:      7 Act:   28 Avg:   30 Max:     110
```


