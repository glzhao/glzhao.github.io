---
layout: post
title: "build kernel"
date: 2014-04-13 21:37:56 +0800
comments: true
categories: kernel
---

# Kernel Build processes

##1. Creating a configuration file

###a. Use a newly created .config

```
$ make config
```

<!-- more -->

###b. Use default .config

```
$ make defconfig
```

###c. Use running OS's .config

####1. Get from procfs 

If the kernel configuration is hard to find, look in the kernel itself. Most distribution kernels 
are built to include the configuration within the /proc filesystem

```
$ zcat /proc/config.gz > .config
```

####2. Get from  /user/src/linux

Almost all distributions provide the kernel configuration files as part of the distribution 
kernel package. Read the distribution-specific documentation for how to find these configurations. 
It is usually somewhere below the /usr/src/linux/ directory tree.

####3. Use oldconfig 

`make oldconfig` takes the current kernel configuration in the .config file, and updates it 
based on the new kernel release. To do this, it prints out all configuration questions, 
and provides an answer for them if the option is already handled in the configuration file. 
If there is a new option, the program stops and asks the user what the new configuration 
value should be set to. After answering the prompt, the program continues on until the 
whole kernel configuration is finished.

```
$ make oldconfig
```

####4. Use silentoldconfig 

`make silentoldconfig` works exactly the same way as oldconfig does, but it does not print 
anything to the screen, unless it needs to ask a question about a new configuration option.

```
$ make silentoldconfig
```

##2. Modifying the configuration file

```
$ make menuconfig
```

or

```
$ make xconfmig
```

or

```
$ make gconfig
```

##3. Building the kernel

a). make

This commond check all changed object files and do the final kernel image link properly

b). make -j $n

It is best to give a number to the -j option that corresponds to twice the number 
of processors in the system. So, for a machine with 2 processors present, use `make -j4` 

c). make -j

The build system will create a new thread for every subdirectory in the kernel tree, 
which can easily cause your machine to become unresponsive and take a much longer time 
to complete the build. Because of this, it is recommended that you always pass a number 
to the -j option

NOTE:
Older kernel versions prior to the 2.6 release required the additional step of make modules
to build all needed kernel modules. That is no longer required now.

##4. Installing and Booting From a Kernel

### Using a Distribution's Installation Scripts

####1. Install kernel modules

Install all the modules which have built and place them in the proper location in the filesystem 
for the new kernel to properly find. Modules are placed in the /lib/modules/KERNEL_VERSION 
directory, where KERNEL_VERSION is the kernel version of the new kernel you have just built.

```
$ make modules_install
```

####2. Install kernel image

Use the following command:

```
$ make install
```

This commond include the following processes:

* The kernel build system will verify that the kernel has been successfully built properly.

* The build system will install the static kernel portion into the /boot directory and name 
this executable file based on the kernel version of the built kernel.

* Any needed initial ramdisk images will be automatically created, using the modules that 
have just been installed during the modules_install phase.

* The bootloader program will be properly notified that a new kernel is present, and it will 
be added to the appropriate menu so the user can select it the next the machine is booted.

### Installing By Hand

####1. Install modules

```
$ make mdoules_install
```

####2. Copy the static kernel image into the /boot directory

a). Get kernel version
	
```
$ make kernelversion
```

b). Copy image and system.map

```
$ cp arch/i386/boot/bzImage /boot/bzImage-KERNEL_VERSION
$ cp System.map /boot/System.map-KERNEL_VERSION
```

c). Modify the bootloader so it knows about the new kernel. This involves editing a configuration file for the bootloader you use.

For GRUB:

```
$ vi /boot/grub/memu.lst
$ reboot
```

For LILO: 
```
$ vi /etc/lilo.conf
$ /sbin/lilo 
```

The second comand `/sbin/lilo` write the configuration changes out to the boot section of the disk.

NOTE:
The kernel version will probably be different for your kernel. Use this value in place of the 
text KERNEL_VERSION in all of the following steps, and you can find the script on the Note 3.

# NOTE:

###1. Permissions 

Do **not** configure or build your kernel with **superuser permissions** enabled!

There have been bugs in the kernel build process in the past, causing some special files 
in the /dev directory to be deleted if the user had superuser permissions while building 
the Linux kernel. 

There are also issues that can easily arise when uncompressing the Linux kernel with 
superuser rights, as some of the files in the kernel source package will not end up with 
the proper permissions and will cause build errors later.

###2. Place for kernel source code 

The kernel source code should also **never** be placed in the /usr/src/linux/ directory.

The location of the kernel that the system libraries were built against, not your new 
custom kernel. Do not do any kernel development under the /usr/src/ directory tree at all, 
but only in a local user directory where nothing bad can happen to the system.

###3. A script used to install the kernel automatically

Here is a handy script that can be used to install the kernel automatically instead of 
having to type the previous commands all the time.

As following:

	#!/bin/sh
	
	# installs a kernel
	make modules_install
	
	# find out what kernel version this is
	for TAG in VERSION PATCHLEVEL SUBLEVEL EXTRAVERSION ; do
		eval `sed -ne "/^$TAG/s/ //gp" Makefile`
	done
	
	SRC_RELEASE=$VERSION.$PATCHLEVEL.$SUBLEVEL$EXTRAVERSION
	
	# figure out the architecture
	ARCH=`grep "CONFIG_ARCH " include/linux/autoconf.h | cut -f 2 -d "\""`
	
	# copy the kernel image
	cp arch/$ARCH/boot/bzImage /boot/bzImage-"$SRC_RELEASE"
	
	# copy the System.map file
	cp System.map /boot/System.map-"$SRC_RELEASE"
	
	echo "Installed $SRC_RELEASE for $ARCH"

###4. Which patch applies to which release

A kernel patch file only will upgrade the source code from one specific release to 
another specific release. Here is how the different patch files can be applied:

* Stable kernel patches apply to the base kernel version. This means that the 2.6.17.10 
patch will only apply to the 2.6.17 kernel release. The 2.6.17.10 kernel patch will not 
apply to the 2.6.17.9 kernel or any other release.

* Base kernel release patches only apply to the previous base kernel version. This means 
that the 2.6.18 patch will only apply to the 2.6.17 kernel release. It will not apply to 
the last 2.6.17.y kernel release, or any other release.

* Incremental patches upgrade from a specific release to the next release. This allows 
developers to not have to downgrade their kernel and then upgrade it, just to switch from 
the latest stable release to the next stable release (remember that the stable release 
patches are only against the base kernel, not the previous stable release.) Whenever 
possible, it is recommended that you use the incremental patches to make your life easier.

###5. Finding the patch

The stable and base kernel patches are located in the same directory structure as the main 
source trees. All incremental patches can be found one level lower, in the incr subdirectory.
So, to find the patch that goes from 2.6.17.9 to 2.6.17.10, we look in the 
/pub/linux/kernel/v2.6/incr directory to find the files we need.

###6. Device discovery

Here are the steps needed to find the driver for a device that has a working driver already bound 
to it:

1. Find the proper sysfs class device that the device is bound to. Network devices are listed in 
/sys/class/net and tty devices in /sys/class/tty. Other types of devices are listed in other 
directories in /sys/class, depending on the type of device.

2. Trace through the sysfs tree to find the module name that controls this device. It will be found 
in the /sys/class/class_name/device_name/device/driver/ module, and can be displayed using the 
readlink and basename applications:

	```
	$ basename `readlink /sys/class/class_name/device_name/device/driver/module`
	```

3. Search the kernel Makefiles for the CONFIG_ rule that builds this module name by using find and grep:

	```
	$ find -type f -name Makefile | xargs grep module_name
	```

4. Search in the kernel configuration system for that configuration value and go to the location 
in the menu that it specifies to enable that driver to be built.

5. Print a list of all of the modules that are needed to control all devices in the system

The script is as following:

	#!/bin/bash
	#
	# find_all_modules.sh
	#

	for i in `find /sys/ -name modalias -exec cat {} \;`; do
	/sbin/modprobe --config /dev/null --show-depends $i ;
	done | rev | cut -f 1 -d '/' | rev | sort -u

