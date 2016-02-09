---
layout:    post
title:     "Darwin on ARMory"
date:      2016-02-08 23:33
---

With [JTAG][usb-armory-jtag] now working on the USB armory I've made progress with one of my goals: porting Darwin on 
ARM to the platform. This is the first Freescale SoC to get a port, as well as my first attempt at writing a platform 
expert for XNU. Currently this boots on the device, however there are issues that still need to be addressed before I 
would consider things to be working properly.

[usb-armory-jtag]: http://embeddedideation.com/2015/12/16/usb-armory-jtag-interface/

Please note, I've tried to make things as accurate as possible, but I'm still learning about many platform components.
Take what you read with a grain of salt and don't rely on it for anything important. Please [drop me a line][contact-page]
if you spot any issues here!

[contact-page]: http://embeddedideation.com/about/


## Configuration

The USB armory uses a Freescale i.MX53 SoC, which has a freely available [reference manual][iMX53RM-pdf]. It has support
in the mainline U-Boot sources. This makes it a fairly ideal target for a port. Darwin on ARM, through GenericBooter,
is designed to take advantage of U-Boot's platform initialization and uImage loading to bootstrap XNU. Some minor 
modifications to U-Boot and GenericBooter are necessary, and a device tree and XNU platform expert must be prepared.

[iMX53RM-pdf]: https://cache.freescale.com/files/32bit/doc/ref_manual/iMX53RM.pdf


### U-Boot

The critical changes made to U-Boot are minimal. LOADADDR is moved higher so that GenericBooter has space to relocate 
the kernel and ramdisk. Also ATAG support is required for passing boot arguments in a form that GenericBooter can present 
to XNU. To ease testing, I increased the boot delay and hardcoded more useful defaults for bootargs and the boot command.


### GenericBooter

XNU requires some additional preparation before booting that U-Boot does not provide. GenericBooter fills this role.
Primarily, it supports img3 parsing and Mach-O loading, as well as device tree preparation. Porting this component 
is simple as it only requires some build tweaks and a basic UART implementation. Since the MMU is disabled in the context 
of GenericBooter, serial output can be achieved by direct access to the UART registers.


### Device Tree

For now, our device tree only describes the basic necessities required to boot and interact with the system. The skeleton
provides a number of bindings that would be filled in by iBoot (and are taken care of by GenericBooter). Other bindings
are for the CPU, SDRAM, interrupt controller, and UART. More should be added for available GPIOs, SPI, I2C, and other 
peripherals.


### Platform Expert

The platform expert provides a layer between XNU and some hardware that is required to complete the booting process.
Some documentation on writing one for Darwin on ARM was provided by winocm, which is now [archived][ios-on-my-toaster].
Several examples are also present in the XNU source under `pexpert/arm/`.

[ios-on-my-toaster]: http://web.archive.org/web/20140625044307/http://winocm.com/research/2013/09/08/ios-on-my-toaster/

The most important part of this is to set up interrupts and a timer in order to drive the system tick. This is a
requirement for operation of the scheduler. A serial console is desirable for logging and debugging system boot, as well 
as providing shell access, so a UART will need to be configured. Additionally, the OS expects the presence of a framebuffer. 
The USB armory does not have a bitmap display, so for now I simply provide a buffer that will appease this requirement.

The interrupt controller needs to be initialized. The i.MX53 supports vectoring interrupts to either the secure or 
non-secure world, but currently I am not taking advantage of TrustZone features. I enable non-secure interrupts and 
start with all interrupt sources disabled.

The system timer is based on the Enhanced Periodic Interrupt Timer (EPIT). I am using the RealView platform expert's
timer configuration as an example for now. For the clock source I chose the high frequency reference clock (AHB clock). 
Based on reading the Clock Control Module registers, this source is running at 133MHz. With the EPIT prescaler set to 133, 
the counter runs at 1MHz. I enable the timer request from the TZIC and the EPIT output compare event so the core will 
receive the interrupt. The decrementing counter is loaded with a value of 5000, so the timer interrupt should fire at 5kHz.

The interrupt service routine checks if the interrupt received is from the system timer, and if so handles it directly.
If it is from a different source, it should be passed onto the IOKit IRQ handler (though this is not yet implemented in
our platform's ISR).

UART is already initialized by U-Boot so I cheat a bit and continue to use what it has already set up for us, which 
seems to work fine. The putc() implementation is essentially identical to the one used in GenericBooter. The getc()
implementation checks for a non-empty Rx FIFO, then reads a character and checks its validity before returning. The function
will timeout and return -1 after a number of iterations. I am not sure why this is necessary yet, but other platform 
experts use include a similar timeout, and the serial console did not seem to work properly without it.


## Building

In order to run this port on the USB armory, you will need a way to build the necessary sources, and a micro SD card. 
Building was performed in a Ubuntu 14.04 LTS VM. 

The version of U-Boot I'm using is a fork from the USB armory project before support was accepted into the main repository.
The changes to my fork are minimal and would probably work fine if integrated into the current mainline, but I haven't
tested this. To build U-Boot, set up the toolchain as described [here][prep-image-guide], then:

~~~
git clone https://github.com/x56/u-boot-usbarmory
cd u-boot-usbarmory
git checkout darwin-on-armory
export CROSS_COMPILE=arm-linux-gnueabihf-
make distclean
make usbarmory_config
make ARCH=arm
~~~

Darwin on ARM provides an SDK and [documentation][building-darwin] on setting up a build environment for XNU and 
GenericBooter. After installing the SDK, you will also need to build the modified dtc from [here][dtc-AppleDeviceTree].

Once your environment is ready, compile the necessary device tree:

~~~
git clone https://github.com/darwin-on-arm/DeviceTrees.git
cd DeviceTrees
make USBarmory_MkI.devicetree
~~~

Get the root filesystem ramdisk:

~~~
git clone https://github.com/darwin-on-arm/ramdisk.git
~~~

Compile the kernel:

~~~
git clone https://github.com/darwin-on-arm/xnu.git
cd xnu
make TARGET_CONFIGS="debug arm imx53" NO_DTRACE_SYMS=YES
~~~

Then compile GenericBooter and build the uImage:

~~~
cd GenericBooter
make menuconfig
# Select "Cortex-A8" for ARM processor target
# Select "Inverse Path USB armory MkI" ARM board target
# Select "iOS-compatible flattened device tree" for Device tree style

# Create kernel IMG3
image3maker -t krnl -f ../xnu/BUILD/obj/DEBUG_ARM_IMX53/mach_kernel -o images/Mach.img3
# Create device tree IMG3
image3maker -t dtre -f ../DeviceTrees/USBarmory_MkI.devicetree -o images/DeviceTree.img3
# Create ramdisk IMG3
image3maker -t rdsk -f ../ramdisk/ramdisk.dmg -o images/Ramdisk.img3
# Cross-compile the bootloader
make CROSS_COMPILE=arm-none-eabi-
~~~

To prepare the SD card for booting, I followed the USB armory [official guide][prep-image-guide], substituting a FAT 
partition to hold the uImage:

~~~
sudo parted /your/sdcard --script mklabel msdos
sudo parted /your/sdcard --script mkpart primary fat32 5M 100%
sudo mkfs.msdos /your/sdcard1
~~~

Once the SD card is formatted, write our U-Boot image to the card:

~~~
sudo dd if=u-boot.imx of=/your/sdcard bs=512 seek=2 conv=fsync
~~~

Then mount the FAT partition and copy the uImage to its root. Insert the SD card into the USB armory and boot!
(You will need to connect a serial cable to the USB armory header, as this is currently the only method for accessing
the console.)

[prep-image-guide]: https://github.com/inversepath/usbarmory/wiki/Preparing-a-bootable-microSD-image
[building-darwin]: https://github.com/darwin-on-arm/wiki/wiki/Building-Darwin
[dtc-AppleDeviceTree]: https://github.com/darwin-on-arm/dtc-AppleDeviceTree


## Future Work

The current i.MX53 timer implementation required some guesswork. While searching the FreeBSD, Linux, and U-Boot source 
trees for guidance on this component I found several typos and unsure comments, so more research and experimentation is 
necessary. The interrupt service routine only handles the EPIT signal; this needs to be fixed to pass requests to IOKit
handlers as more hardware support is added.

The XNU port itself has many outstanding issues to address including random file corruption and hangs when booting to 
multi-user mode.

I am interested in exploring more ARM platform features. To that end, I may try to get XNU running alongside a secure 
monitor on the i.MX53. Also, ARMv8 support is incomplete. I have a Nexus 9 that would make an interesting target, though 
I may save myself from ripping out some hair by acquiring a more open (and documented) development platform.

