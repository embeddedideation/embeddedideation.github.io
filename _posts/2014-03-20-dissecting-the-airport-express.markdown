---
layout:    post
title:     "Dissecting the AirPort Express"
date:      2014-03-20 22:45:00
permalink: /2014/03/dissecting-the-airport-express/
---

I became interested in exploring the AirPort family of routers a few weeks ago, after learning of an easily accessible
serial port on the AirPort Express 2nd Generation N router (thanks to Chris, a.k.a. capin for sharing the discovery).
While searching for information about this device, and the rest of the AirPort family, I discovered that there really
isn't that much out there. There are just a few blog posts and teardowns available on the web with virtually no discussion
of custom firmware builds, hacks, or other in depth analyses. Out of the desire to explore something yet unknown, I picked
up a couple of routers. This post documents some of my findings and thoughts on what could be done next with these devices.
I currently only possess the 2nd Generation N model of the AirPort Express; more posts may come when I've acquired more
hardware to investigate.

## Disassembly

Getting inside the AirPort Express itself was tricky. My first disassembly cost several cheap plastic pry tools and a bit
of blood (one of the spudgers broke while prying and stabbed my thumb). The second device I tried came apart with much less
trouble with the help of an iSesamo opening tool. The case comprises two parts, which are secured together with plastic
clips. I was unable to release some of the clips, particularly those in the corners, without breaking them. However, it
seems only a few clips are necessary to hold the bottom panel on securely when reassembling the unit. A quick disassembly
guide follows:

There are two tabs on the bottom panel on the side closest to the rear ports. Release them first by prying
the top case away from the bottom panel next to each tab:

![Bottom panel tabs]({{ site.url }}/assets/a1392-bottom-case-tabs.jpg)
_Two tabs on the bottom panel_

The rest of the bottom panel is secured by clips around the edges. Insert the pry tool adjacent to the clip. This time, pry
the inner edge of the bottom panel inwards, to release it from the clip protruding inward from the top case. Work around
the perimeter of the panel, releasing the clip in each spot indicated:

![Tab locations]({{ site.url }}/assets/a1392-case-clip-locations.jpg)
_The remaining seven tabs are evenly spaced around the perimeter_

The logic board and the power supply are the only two modules inside. The logic board is held down with Torx T8 screws.
There are two Torx T6 screws fastening the power cord connector in place. Removal of the logic board is not strictly
necessary to access the debug header, though it may make soldering to the test pads easier.

## Hardware

I found two [previous][prev-teardown-1] [teardowns][prev-teardown-2] of the 2nd Generation AirPort Express.
Noticably absent was one from iFixit, who performed teardowns of several other devices in the AirPort family. The main
components on the logic board are as follows:

  * Atheros AR9344 SoC (Application Processor and 2.4GHz radio)
  * Atheros AR9582 (5GHz radio)
  * Nanya NT5TU32M16DG-AC (64MB DDR2 SDRAM)
  * Winbond W25Q128BV (16MB SPI NOR flash)
  * AKM AK4430ET (192kHz 24-Bit Stereo DAC)

The AP contains a MIPS 74Kc core in a big endian configuration. Substituted for the Hynix memory found on other devices
is the Nanya NT5TU32M16DG-AC. One thing overlooked by other teardowns was the SPI flash. It was obscured by a part of one
of the EMI shield anchors; with the anchor removed, I was able to view the package:

![Winbond SPI flash]({{ site.url }}/assets/a1392-spi-flash.jpg)
_Winbond 25Q129BVEG 128Mbit SPI Flash_

[prev-teardown-1]: http://weblog.rogueamoeba.com/2012/06/19/airport-express-disassembly/
[prev-teardown-2]: http://www.smallnetbuilder.com/wireless/wireless-features/31794-inside-story-apple-airport-express-2012-and-wd-my-net-n900

## Serial Console

Upon removal of the bottom panel, there is a debug header visible in the corner, adjacent to the audio jack. The header
layout nearly corresponds to the compact TI 20 pin connector for JTAG on certain ARM devices, except the two columns of
pads are offset slightly:

![Debug header close-up]({{ site.url }}/assets/a1392-debug-header.jpg)
_Close-up image of the debug header_

Two of the pads are connected to the onboard low-speed UART. I used a FT232RL adapter to interface with the port, and
connected with screen (115200 baud, 8 data bits, no parity, 1 stop bit, no flow control). A root shell is available
immediately requiring no authentication.

## Locating JTAG

A closer inspection of the debug header (both visual and with a continuity tester) allowed me to identify several more
of the pads as VCC, ground, or unconnected. At this point, six pads remained unidentified. MIPS EJTAG commonly utilizes
six signals (TDI, TDO, TCK, TMS, nTRST, and nSRST), so I figured this was a likely candidate for the leftover unknowns.
First, I tried using an Arduino Mega with [JTAGenum][jtagenum] to differentiate
the signal lines automatically. When either of two out of the six pads were probed, the router would reset. Unfortunately,
probing the other four together yielded no results. Referencing the AR9344 datasheet next, I located the pads on the BGA
corresponding to the four GPIO lines (GPIO[3:0]) that carry the JTAG signals. I decided to sacrifice one of my two test
devices to test for continuity between the BGA pads and the debug header pads. The BGA removal went rather poorly, and the
tracings underneath were damaged enough that I was unable to locate any connections between a set of pads. My final idea
was to manipulate the GPIO registers to try to map out the connection between the header and SoC. CFE (the bootloader;
more on that later) has built-in functions to read and write at arbitrary memory addresses. According to the datasheet,
the GPIO registers are mapped at 0x18040000. Dumping 0x6F bytes at that location gives us the current state of these 
registers:

[jtagenum]: http://deadhacker.com/2010/02/03/jtag-enumeration/

~~~
CFE> d 0x18040000 0x6F  
18040000: 0081331B 0003BA30 00028800 00000000 ..3....0........  
18040010: 00000000 00000000 00000000 00000000 ................  
18040020: 00000000 00000000 000F8000 00000000 ................  
18040030: 0B0A0914 00180000 00000000 0000004D ...............M  
18040040: 00000000 00000908 00000000 00000000 ................  
18040050: 00000C0B 00000000 00000000 00000000 ................  
18040060: 00000000 0D0F110E 00000000 00000040 ...............@  
*** command status = 0
~~~

After connecting a logic analyzer to the unknown pads, I set the DISABLE\_JTAG bit, configured GPIO[3:0] as outputs, and
set the outputs high. One of the four unknown signals went from low to high (corresponding to the TDO pin). I performed a
capture while toggling the output for confirmation:

![Capture while toggling TDO]({{ site.url }}/assets/a1392-saleae-tdo-toggle.png)
_Captured while toggling the output level of the TDO line_

Despite configuring all four GPIOs as outputs, only TDO responded with any change of output. To test the other three pads,
I pulled each of them up by connecting them to VCC through a 10kΩ resistor. I set them as inputs this time, dumped the GPIO
registers, and observed the signals read high. By testing each individually, I was able to map the remaining three pads. Of
the two pads that cause the device to reset when pulled low, I have not figured out how to identify which is nTRST and
which is nSRST. I invite a reader with any thoughts on how to differentiate the two to leave a comment. Here is a labeled
pinout of the debug header:

![Debug Header Pinout]({{ site.url }}/assets/a1392-header-pinout.jpg)
_Pinout of the AirPort Express N 2nd Generation router debug header_

## Firmware

Persistent storage is composed of five partitions on the 16MB flash. Three of the partitions contain executable code, while
the other two contain configuration data. The partitions are as follows:

  * boot - Common Firmware Environment (CFE) bootloader
  * scfg - stored firmware version number, radio calibration data, and some device specific identifiers
  * dcfg - AirPort configuration data and SSH host keys
  * primary - gzboot image containing the kernel and root filesystem
  * secondary - backup gzboot image

#### CFE

CFE is used for the primary bootloader. The bootloader initializes the system and reads the scfg partition to populate some
environment variables. The CFE console can be accessed through the serial console by pressing a key shortly after boot.
Several useful commands are available including the aforementioned memory read/write commands. After a one second timeout
the default boot commences.

#### Kernel

The device runs a NetBSD 4.0 kernel. The kernel (along with the root filesystem) is stored as a gzboot image. The gzboot
image is a self-loading gzip compressed image, similar to a Linux zImage. The stub loader is executed directly from SPI
flash, which decompresses and loads the image contents to DRAM, then jumps to the kernel entrypoint.

#### Root Filesystem

The root filesystem comprises one monolithic binary as well as assorted scripts and configuration files. All of the system
binaries are, in fact, hard links to this binary, named "crunchprog". Contained in crunchprog are standard system and
networking tools, as well as several proprietary daemons for Apple services. These include Bonjour, Back to My Mac,
AirPrint, AirPlay, and AirPort Utility configuration. The dcfg partition is mounted at /mnt/Flash, and provides keys and
configuration data used for device configuration on startup, as well as for updates from the host through AirPort Utility.

#### Basebinary

Firmware updates are distributed from Apple as basebinary files. The basebinary is simply a container and its contents are
encrypted. The file header contains a magic string, an integer representing the product ID of the device that the firmware
targets, and several more bytes of unknown function. The Adler-32 checksum of the basebinary is appended to the file.

AirPort Utility checks [http://apsu.apple.com/version.xml](http://apsu.apple.com/version.xml) for updates. When updating
the device's firmware, both the basebinary and a signature file are downloaded to the host. The firmware update is verified
before it is sent to the device. It is verified again on the device, then unpacked and written to flash storage.

## Next Steps

Finding JTAG was the goal to reach before stopping to publish a summary of my research. With the serial console and JTAG
interface available, it is possible to develop and hack on this AirPort Express with essentially no fear of bricking the
device (provided backups of flash contents are acquired first, in the case that CFE and both gzboot images are damaged).
This opens a few avenues for further research.

Access to the device over something other than a serial interface would be very helpful for tinkering with the device. An
SSH server is available, however not configured to run by default. Setting up SSH over the serial console is possible
without copying anything to the device, either using ssh-keygen or borrowing the SSH host keys present at /mnt/Flash, and
providing a basic sshd_config (and also adjusting the firewall with pfctl). This setup is not persistent because the root
filesystem is backed by a ramdisk refreshed at each boot with the filesystem image stored inside the gzboot image.
Persistent access to the device will require modifying the ramdisk contents with a new configuration and writing them in a
new gzboot image to flash.

Considering the variety of proprietary services running on the device, an audit of the kernel and crunchprog binaries
should be performed. Many of the services allow local or remote access to the device using undocumented protocols. A
cursory examination of the system shows a lack of hardening, with the likely consequence of compromising one of many
daemons being full control of the target device.

## Wiki

In order to bring together those interested in hacking the AirPort family of devices, capin established
[The AirPort Wiki](http://www.theairportwiki.com/). The goal of this wiki is to document device internals, and lay the
groundwork for adding new features to the device, or possibly porting a custom firmware. More technical details of my
research will be added to the wiki as I have the opportunity to assemble and publish them. There is an associated IRC
channel, #theairportwiki on Freenode, for the discussion of hacking AirPort devices.
