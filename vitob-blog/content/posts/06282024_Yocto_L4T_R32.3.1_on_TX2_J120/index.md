+++
title = 'Yocto L4T R32.3.1 on TX2 J120'
date = 2024-06-29T21:06:18-07:00
draft = false
toc = true
+++


Finally we are here, with TX2 sitting on the J120 carrier board, we are ready to flash the setup with customized patched Yocto. 

## Yocto on TX2-J120 with Jetpack4.3

### Meta-tegra, Poky, meta-tx2-jp43

Very similar to previous blog article [Yocto on TX2 with Jetpack4.3](ihttps://jinchenglee.github.io/posts/06102024_yocto_on_tx2/), we set up meta-tegra, poky, except that we need to setup layer meta-tx2-jp43 to a special branch for J120.

```
cd ~ && mkdir yocto-tx2-jp43-j120 && cd yocto-tx2-jp43-j120

git clone https://github.com/OE4T/meta-tegra.git 
cd meta-tegra
git branch -a
git checkout dunfell-l4t-r32.3.1

cd ~/yocto-tx2-jp43-j120
git clone https://github.com/yoctoproject/poky.git
cd poky
git branch -a
git checkout dunfell

cd ~/yocto-tx2-jp43-j120
git clone https://github.com/jinchenglee/meta-tx2-jp43.git
cd meta-tx2-jp43
git branch -a
git checkout dev/tx2-j120-jp43  # Special branch for TX2 on J120

cd ~/yocto-tx2-jp43-j120
source poky/oe-init-build-env
```

### Local.conf and bblayers.conf

* Changes in local.conf
```
MACHINE ??= "jetson-tx2"

MACHINE_ESSENTIAL_EXTRA_RRECOMMENDS += "kernel-modules"

DL_DIR ?= "/home/${USER}/Yocto/downloads"

SSTATE_DIR ?= "/home/${USER}/Yocto/sstate_dir"

PACKAGE_CLASSES ?= "package_deb"  # for apt (?)
EXTRA_IMAGE_FEATURES ?= "debug-tweaks tools-sdk package-management"

IMAGE_CLASSES += "image_types_tegra"

PREFERRED_VERSION_python3 = "3.8%"
PREFERRED_VERSION_python3-native = "3.8%"

BB_NUMBER_THREADS = '11'
PARALLEL_MAKE = '-j11'

IMAGE_INSTALL_append = " glfw cmake vim tmux"
```

* Changes in bblayers.conf
```
BBLAYERS ?= " \
  /home/${USER}/yocto-tx2-jp43/meta-tegra \
  /home/${USER}/yocto-tx2-jp43/poky/meta \
  /home/${USER}/yocto-tx2-jp43/poky/meta-poky \
  /home/${USER}/yocto-tx2-jp43/poky/meta-yocto-bsp \
  /home/${USER}/yocto-tx2-jp43/meta-tx2-jp43 \
  /home/${USER}/yocto-tx2-jp43/meta-openembedded/meta-oe \ # cmake or gflw seems require this.
  "
```

### Build Image

```
> bitbake  core-image-sato-sdk    # Or core-image-sata-dev, core-image-minimal etc.
```

### Deploy

Script:
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

```
sudo ./deploy.sh core-image-sato-sdk jetson-tx2
```

This image should have both real-time patch and RealSense R200 supports. 
```
root@jetson-tx2:~# uname -a
Linux jetson-tx2 4.9.140-rt93-l4t-r32.3.1+ga0004d2ad6a4 #1 SMP PREEMPT RT Thu Jan 28 22:42:33 UTC 2021 aarch64 aarch64 aarch64 GNU/Linux
```


## Build RTIMULib2

My J120 carrier board has a on-board [9-axis (Gyro, Accelerometer, Compass) MPU-9250](https://invensense.tdk.com/products/motion-tracking/9-axis/mpu-9250/). Here we show how the demo app can be compiled and run on the Yocto-linux just flashed. 

### Get RTIMULIB2

There's no git installed on the flashed TX2-J120. Let's get the zipped version from [github:RTIMULib2](https://github.com/andrasj/RTIMULib2), then `rcp` it over to device.
```
rcp RTIMULib2-master.zip root@<device ip addr>:/home/root/
```

### Compile RTIMULib
```
root@jetson-tx2:~/RTIMULib2-master# cd RTIMULib/
root@jetson-tx2:~/RTIMULib2-master/RTIMULib# ls
CMakeLists.txt	RTFusionKalman4.cpp  RTIMUAccelCal.cpp	RTIMUHal.h	  RTIMULibDefs.h     RTIMUSettings.h
IMUDrivers	RTFusionKalman4.h    RTIMUAccelCal.h	RTIMULIB LICENSE  RTIMUMagCal.cpp    RTMath.cpp
RTFusion.cpp	RTFusionRTQF.cpp     RTIMUCalDefs.h	RTIMULib.h	  RTIMUMagCal.h      RTMath.h
RTFusion.h	RTFusionRTQF.h	     RTIMUHal.cpp	RTIMULib.pri	  RTIMUSettings.cpp  build
root@jetson-tx2:~/RTIMULib2-master/RTIMULib# mkdir build && cd build
root@jetson-tx2:~/RTIMULib2-master/RTIMULib/build# cmake ..
root@jetson-tx2:~/RTIMULib2-master/RTIMULib/build# make -j
```

### Compile Application
Some of the applications need Qt4 library that doesn't exist on the Yocto built we prepared, so we will first comment those off in <tot>/Linux/CMakeLists.txt: 
```
OPTION(BUILD_GL "Build RTIMULibGL" OFF)                                                 
OPTION(BUILD_DRIVE "Build RTIMULibDrive" ON)                                            
OPTION(BUILD_DRIVE10 "Build RTIMULibDrive10" ON)                                        
OPTION(BUILD_DRIVE11 "Build RTIMULibDrive11" ON)                                        
OPTION(BUILD_CAL "Build RTIMULibCal" OFF)                                               
OPTION(BUILD_DEMO "Build RTIMULibDemo" OFF)                                             
```

Then compile:
```
root@jetson-tx2:~/RTIMULib2-master/Linux/build# pwd
/home/root/RTIMULib2-master/Linux/build
root@jetson-tx2:~/RTIMULib2-master/Linux/build# cmake ..
root@jetson-tx2:~/RTIMULib2-master/Linux/build# make -j
```

On our J120 board (or setting in device tree?), the MPU-9250 is on SPI bus 1 device 0 (spidev1.0). A related [forum disucssion](https://forums.developer.nvidia.com/t/running-rtimulib2-on-tx1-with-auvidea-j120-carrier-board/51052) confirmed this as well. So we have to modify RTIMULib.ini configuration before running apps. 

```
# IMU type -                                                           
#   ...
#   7 = InvenSense MPU-9250                               
#   8 = STM L3GD20H + LSM303DLHC                          
#   ...
IMUType=7                       
                                                            
# Is bus I2C: 'true' for I2C, 'false' for SPI                
BusIsI2C=false                                               
                
# SPI Bus (between 0 and 7)
SPIBus=1                   
                              
# SPI select (between 0 and 1)
SPISelect=0                  
```

Run the app (guess the MPU is not calibrated, so the data are not trust-worthy):
```
root@jetson-tx2:~/RTIMULib2-master/Linux/build/RTIMULibDrive11# vi RTIMULib.ini 
root@jetson-tx2:~/RTIMULib2-master/Linux/build/RTIMULibDrive11# sudo ./RTIMULibDrive11 
Settings file RTIMULib.ini loaded
Using fusion algorithm RTQF
Detected MS5611 at standard address
Detected HTU21D at standard address
min/max compass calibration not in use
Ellipsoid compass calibration not in use
Accel calibration not in use
MPU-9250 init complete
Sample rate 0: : roll:179.025433, pitch:-0.544863, yaw:43.259560
Pressure: -334.6, height above sea level:  nan, temperature:  0.0, humidity:  0.0
Sample rate 0: : roll:178.940793, pitch:-0.527068, yaw:43.185848
Pressure: -268.4, height above sea level:  nan, temperature: -46.8, humidity:  0.0
Sample rate 0: : roll:178.934127, pitch:-0.513112, yaw:43.088907
Pressure: -268.4, height above sea level:  nan, temperature: -46.8, humidity: -16.8
Sample rate 0: : roll:178.963428, pitch:-0.509524, yaw:42.994914
Pressure: -268.4, height above sea level:  nan, temperature: -46.8, humidity: -16.8
Sample rate 85: : roll:179.010133, pitch:-0.507106, yaw:42.906466
Pressure: -268.4, height above sea level:  nan, temperature: -46.8, humidity: -16.8
Sample rate 85: : roll:179.057616, pitch:-0.509052, yaw:42.830696
Pressure: -268.4, height above sea level:  nan, temperature: -46.8, humidity: -16.8
...
```

