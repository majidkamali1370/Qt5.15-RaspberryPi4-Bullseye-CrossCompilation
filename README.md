# Cross compiling Qt 5.15.2 for RaspberryPi 4 device and Bullseye OS

Bellow is a tested method of cross-compiling Qt5.15.2 for RaspberryPi 4 and Bullseye.

**Note:** Commands starting with a `$` sign are bash scripts. Try not to copy the `$` signs :)

## Host specifications

 - Intel Core i7
 - 16Gb RAM
 - Arch linux OS (x64)

## Target specifications

 - RaspberryPi 4 (4Gb)
 - RaspberryPi Bullseye OS (x64)

## Step 1: Download the RaspberryPi OS image [On Host]

1. Download the RaspberryPi Bullseye lite OS from [here](https://www.raspberrypi.com/software/).
2. Extract OS image from archive.
3. Copy OS image content to SD card. First find SD card device with `lsblk` command. Then copy image content with `dd` command to **unmounted** SD card.
```bash
$ sudo dd if=2021-10-30-raspios-bullseye-armhf-lite.img of=/dev/mmcblk0 status=progress conv=fsync bs=100M
```

## Step 2: Setup wifi [On Host]

Mount `/boot` partition of SD card and create a file, named `wpa_supplicant.conf` with following content:

```
country=GB
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
	ssid="NETWORK-NAME"
	psk="NETWORK-PASSWORD"
	priority=1
}
```

Change `ssid` to your wifi name and `psk` to your wifi password. See ref [2] for a more complete file template.

## Step 3: Configure the RaspberryPi [On Pi]

Put SD card in RaspberryPi slot and turn that on. On first boot, bullseye resizes the `/root` partition to occupy whole remaining SD capacity, and then reboots again.

Then enter `pi` as login user and `raspberry` as login password. You can change both of these later.

Type following command in prompt:

```bash
$ sudo raspi-config
```

From `3 Interface Options -> I2 SSH -> Enable`, enable SSH. Also there are many other useful options in these menus. Note that unlike older versions of Raspbian OS, there is no option for setting GL driver in Bullseye. Because KMS driver is the default and enabled (unlike older Raspbian OSes).

## Step 4: Enable development sources [On Pi]

Open `/etc/apt/sources.list` file and uncomment the last line.

```bash
$ sudo nano /etc/apt/sources.list
```
```bash
deb-src http://raspbian.raspberrypi.org/raspbian/ buster main contrib non-free rpi
```

Now update the system

```bash
$ sudo apt-get update
$ sudo apt-get dist-upgrade
$ sudo reboot
```

## Step 5: Enable rsync with elevated rights [On Pi]

Later in this guide, we will be using the `rsync` command to sync files between the Host PC/Laptop and the Raspberry Pi. For some of these files, root rights (i.e. sudo) is required internally.

You can do this with a single terminal command as follows:

```bash
echo "$USER ALL=NOPASSWD:$(which rsync)" | sudo tee --append /etc/sudoers
```

## Step 6: Install the required development packages [On Pi]

```bash
$ sudo apt install build-essential cmake unzip pkg-config gfortran
$ sudo apt build-dep qt5-qmake libqt5gui5 libqt5webengine-data libqt5webkit5 libudev-dev libinput-dev libts-dev libxcb-xinerama0-dev libxcb-xinerama0 gdbserver
$ sudo apt install libxcb-randr0-dev libxcb-xtest0-dev libxcb-shape0-dev libxcb-xkb-dev
```

Following packages are optional (for multimedia and bluetooth)

```bash
# additional (multimedia) packages
$ sudo apt install libjpeg-dev libpng-dev libtiff-dev libavcodec-dev libavformat-dev libswscale-dev libv4l-dev libxvidcore-dev libx264-dev openjdk-8-jre-headless
# audio packages
$ sudo apt install libopenal-data libsndio7.0 libopenal1 libopenal-dev pulseaudio
# bluetooth packages
$ sudo apt install bluez-tools libbluetooth-dev
# gstreamer (multimedia) packages
$ sudo apt install libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-pulseaudio libgstreamer1.0-dev  libgstreamer-plugins-base1.0-dev
```

## Step 7: Create Qt libraries directory [On Pi]

Qt libraries will be deployed to `/usr/local/qt5.15`, so create a directory with following commands.

```bash
sudo mkdir /usr/local/qt5.15
sudo chown -R pi:pi /usr/local/qt5.15
```

## Step 8: SSymlinker [On Pi]

Download SSymlinker script for setting required symbolic links.

```bash
$ wget https://raw.githubusercontent.com/abhiTronix/raspberry-pi-cross-compilers/master/utils/SSymlinker
$ sudo chmod +x SSymlinker
$ ./SSymlinker -s /usr/include/arm-linux-gnueabihf/asm -d /usr/include
$ ./SSymlinker -s /usr/include/arm-linux-gnueabihf/gnu -d /usr/include
$ ./SSymlinker -s /usr/include/arm-linux-gnueabihf/bits -d /usr/include
$ ./SSymlinker -s /usr/include/arm-linux-gnueabihf/sys -d /usr/include
$ ./SSymlinker -s /usr/include/arm-linux-gnueabihf/openssl -d /usr/include
$ ./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crtn.o -d /usr/lib/crtn.o
$ ./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crt1.o -d /usr/lib/crt1.o
$ ./SSymlinker -s /usr/lib/arm-linux-gnueabihf/crti.o -d /usr/lib/crti.o
```

## Step 9: Update host machine and install required packages [On Host]

Ubuntu:
```bash
$ sudo apt update
$ sudo apt dist-upgrade
$ sudo apt install build-essential cmake unzip gfortran
$ sudo apt install gcc git bison python gperf pkg-config gdb-multiarch wget
```

Archlinux:
```bash
$ sudo pacman -Syu
$ sudo pacman -S base-devel cmake unzip gfortran
$ sudo pacman -S gcc git bison python gperf pkg-config wget
```

At the time of writing of this text, `gdb-multiarch` package is in `AUR`.

## Step 10: Setup directory structure [On Host]

Create following directory structure (Although I don't recommend puting `rpi-qt` directory in home directory).

```bash
$ sudo mkdir ~/rpi-qt
$ sudo mkdir ~/rpi-qt/build
$ sudo mkdir ~/rpi-qt/tools
$ sudo mkdir ~/rpi-qt/sysroot
$ sudo mkdir ~/rpi-qt/sysroot/usr
$ sudo mkdir ~/rpi-qt/sysroot/opt
$ sudo chown -R 1000:1000 ~/rpi-qt
$ cd ~/rpi-qt
```

## Step 11: Download and extract Qt sources [On Host]

Download and extract Qt5.15.2 sources from [qt.io](qt.io) website.

```bash
$ sudo wget http://download.qt.io/archive/qt/5.15/5.15.2/single/qt-everywhere-src-5.15.2.tar.xz
$ sudo tar xfv qt-everywhere-src-5.15.2.tar.xz 
```

## Step 12: Patch Qt mkspecs [On Host]

We need to slightly modify the a mkspec file within the source files to allow us to use our cross compiler.

```bash
$ cp -R qt-everywhere-src-5.15.2/qtbase/mkspecs/linux-arm-gnueabi-g++ qt-everywhere-src-5.15.2/qtbase/mkspecs/linux-arm-gnueabihf-g++
$ sed -i -e 's/arm-linux-gnueabi-/arm-linux-gnueabihf-/g' qt-everywhere-src-5.15.2/qtbase/mkspecs/linux-arm-gnueabihf-g++/qmake.conf
```

Also we need to change two other files. **This is an important step.** First go to the following directory:

```bash
$ cd ~/rpi-qt/qt-everywhere-src-5.15.2/qtbase/src/plugins/platforms/eglfs/deviceintegration/eglfs_brcm/
```

In `eglfs_brcm.json` file, remove `eglfs_brcm` key from `Keys` list.

```json
{
    "Keys": [  ]
}
```

Go up one directory.

```bash
$ cd ..
```

Open `deviceintegration.pro` file and comment-out or remove the line which contains `eglfs_brcm`.

```
...
# qtConfig(eglfs_brcm): SUBDIRS += eglfs_brcm
...
```

Then go back to the base folder:

```bash
$ cd ~/rpi-qt/
```

## Step 13: Download and extract the cross-compiler [On Host]

First change current working directory to `tools`.

```bash
$ cd ~/rpi-qt/tools
```

Download GCC 10.3.0 cross compiler for RaspberryPi on linux host.

```bash
$ wget https://sourceforge.net/projects/raspberry-pi-cross-compilers/files/Raspberry%20Pi%20GCC%20Cross-Compiler%20Toolchains/Bullseye/GCC%2010.3.0/Raspberry%20Pi%203A%2B%2C%203B%2B%2C%204/cross-gcc-10.3.0-pi_3%2B.tar.gz
```

Then Extract compiler files.

```bash
$ tar xf cross-gcc-*.tar.gz
```

## Step 14: Sync RaspberryPi sysroot [On Host]

Go back to the base folder.

```bash
$ cd ~/rpi-qt
```

Sync required libraries and includes from RaspberryPi to host. Replace `192.168.1.47` with ip address of RaspberryPi.

```bash
$ rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/lib sysroot
$ rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/usr/include sysroot/usr
$ rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/usr/lib sysroot/usr
$ rsync -avz --rsync-path="sudo rsync" --delete pi@192.168.1.47:/opt/vc sysroot/opt
```

## Step 15: Fix symbolic links [On Host]

Download following python script and fix symbolic links.

```bash
$ wget https://raw.githubusercontent.com/abhiTronix/rpi_rootfs/master/scripts/sysroot-relativelinks.py
$ sudo chmod +x sysroot-relativelinks.py
$ ./sysroot-relativelinks.py sysroot
```

## Step 16: Configure Qt Build [On Host]

Go to the `build` directory.

```bash
$ cd ~/rpi-qt/build
```

Then run following commands for starting Qt configuration.

```bash
$ CROSS_COMPILER_LOCATION=$HOME/rpi-qt/tools/cross-pi-gcc-*
$ ../qt-everywhere-src-5.15.2/configure -release -opengl es2  -eglfs -device linux-rasp-pi4-v3d-g++ -device-option CROSS_COMPILE=$(echo $CROSS_COMPILER_LOCATION)/bin/arm-linux-gnueabihf- -sysroot ~/rpi-qt/sysroot/ -prefix /usr/local/qt5.15 -extprefix ~/rpi-qt/qt5.15 -opensource -confirm-license -skip qtscript -skip qtwayland -skip qtwebengine -nomake tests -make libs -pkg-config -no-use-gold-linker -v -recheck -L$HOME/rpi-qt/sysroot/usr/lib/arm-linux-gnueabihf -I$HOME/rpi-qt/sysroot/usr/include/arm-linux-gnueabihf
```

Skiping unneeded Qt modules and not making tests and examples, reduce build time significantly. Note the `-extprefix` parameter value, this is the directory which `make install` will put final libraries and includes in.

**Note:** You might encounter compilation errors in this stage. Each time compiler complains about not finding `numeric_limits` class. Just add `#include <limits>` on top of file with error (next to other includes). Then retry same command without cleaning anything, for the next same error :) . I encountered these errors in 3 or 4 different files. After that, configure will complete successfully.

## Step 17: Check configure output [On Host]

After configure finishes, check following lines in the output.

```bash
Configure summary:

Building on: linux-g++ (x86_64, CPU features: mmx sse sse2)
Building for: devices/linux-rasp-pi4-v3d-g++ (arm, CPU features: neon)
Target compiler: gcc 10.3.0
Configuration: cross_compile compile_examples enable_new_dtags largefile neon precompile_header shared shared rpath release c++11 c++14 c++17 c++1z concurrent dbus reduce_exports stl
```

## Step 18: Check if EGLFS enabled [On Host]

Check EGLFS section in configure output and confirm following values.

```bash
QPA backends:
  EGLFS .................................. yes	[SHOULD BE YES]
  EGLFS details:
    EGLFS OpenWFD ........................ no
    EGLFS i.Mx6 .......................... no
    EGLFS i.Mx6 Wayland .................. no
    EGLFS RCAR ........................... no
    EGLFS EGLDevice ...................... yes	[SHOULD BE YES]
    EGLFS GBM ............................ yes
    EGLFS VSP2 ........................... no
    EGLFS Mali ........................... no
    EGLFS Raspberry Pi ................... yes	[SHOULD BE NO]
    EGLFS X11 ............................ yes
```

The `EGLFS Raspberry Pi` line actually must be `no`, but because brcm libraries are still available in RaspberryPi 4, Qt tries to locate and use them. We did `Step 12` to prevent Qt from using these libs.

## Step 19: Build Qt [On Host]

While you're still in `build` directory

```bash
$ make -j4
```

Change number `4` based on your computer threads count. Be careful that setting high values, results in overheating.

Building Qt takes time and depends on how many modules you `skip`ped in `configure` command.

## Step 20: Install Qt [On Host]

Install Qt libraries to host qt5.15 directory.

```bash
$ make install
```

## Step 21: Deploy Qt libraries to RaspberryPi [On Host]

Sync host Qt libraries directory with RaspberryPi `/usr/local/qt5.15`. Change `192.168.1.47` with your RaspberryPi ip.

```bash
$ cd ~/rpi-qt
$ rsync -avz --rsync-path="sudo rsync" qt5.15 pi@192.168.1.47:/usr/local
```

## Additional step: Configure QtCreator

See ref[4] for configuring QtCreator when building and deploying application to RaspberryPi.

## References

 1. [https://www.interelectronix.com/qt-515-cross-compilation-raspberry-compute-module-4-ubuntu-20-lts.html](https://www.interelectronix.com/qt-515-cross-compilation-raspberry-compute-module-4-ubuntu-20-lts.html)
 2. [https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4](https://github.com/UvinduW/Cross-Compiling-Qt-for-Raspberry-Pi-4)
 3. [https://github.com/abhiTronix/raspberry-pi-cross-compilers/blob/master/QT_build_instructions.md](https://github.com/abhiTronix/raspberry-pi-cross-compilers/blob/master/QT_build_instructions.md)
 4. [https://www.interelectronix.com/configuring-qt-creator-ubuntu-20-lts-cross-compilation.html](https://www.interelectronix.com/configuring-qt-creator-ubuntu-20-lts-cross-compilation.html)
