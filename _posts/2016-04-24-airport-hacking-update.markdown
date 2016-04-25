---
layout:    post
title:     "AirPort Hacking Update"
date:      2016-04-24 18:00
---

Since writing [an article][airport-post] on the subject about 2 years ago, I've continued to poke at AirPort devices in 
my spare time. This will most likely be my final post on the subject, so I'll provide some code I wrote and a brief 
description of discoveries made along the way.

[airport-post]: http://embeddedideation.com/2014/03/dissecting-the-airport-express/


## AirPyrt Tools

To facilitate interacting with AirPort devices I implemented several parts of the management protocol in Python. This
implementation is incomplete but provides a starting point for further research or developing your own client. These tools
aren't suitable for production, most importantly due to the lack of SRP authentication and the encryption protocol used in 
new firmwares. This means the admin password and all session data goes over the network unencrypted.

The code is [available on GitHub][airpyrt-tools-repo]. The README contains a list of things I didn't get around to finishing
for the initial release.

[airpyrt-tools-repo]: https://github.com/x56/airpyrt-tools


### cflbinary Property Lists

Several protocol messages and NVRAM properties are found in a serialized format. This format is another, previously
undocumented type of plist. It is similar to the [bplist format][cfbinaryplist-source] in that variable size objects are 
demarcated by an identifier and in some cases a size value (see `OBJECT TABLE` starting at line 244). It does not support
key/object references or set types however, and objects are concatenated (or in the case of dicts, nested).

[cfbinaryplist-source]: http://opensource.apple.com/source/CF/CF-1153.18/CFBinaryPList.c

The [Python implementation][cflbinary-source] handles composing and parsing cflbinary plists. It does not support date 
objects, but I did not find that object type in any captures (though it is supported by the client and server).

[cflbinary-source]: https://github.com/x56/airpyrt-tools/blob/master/acp/cflbinary.py


### Basebinary Firmware

Firmware updates are available from [http://apsu.apple.com/version.xml](http://apsu.apple.com/version.xml). I wrote
a [simple script][grab-firmwares] to parse this plist and download all available firmwares.

[grab-firmwares]: https://gist.github.com/x56/7790380ea7a8980c69c3

Most of the basebinary containers hold encrypted data. The cipher used is AES in CBC mode, with a 128 bit key differing 
per device model. An obfuscated form of the key is compiled into crunchprog. The firmware data itself is encrypted in 
0x8000 byte chunks.

The released code includes [an implementation][basebinary-py] to decrypt basebinary files as well as a few keys for 
devices that I had on hand. One caveat, the tool must be run twice on a given file to extract the internal gzimage (as
all available downloads contain two nested containers with only the inner one encrypted).

[basebinary-py]: https://github.com/x56/airpyrt-tools/blob/master/acp/basebinary.py


### SRP Authentication and Session Encryption

The current management protocol version uses SRP to authenticate to a server without sending the admin password over the 
network. The shared secret generated from this process is fed into PBKDF with a different salt and iteration count on the
client and server. This provides two unidirectional session keys for client-server communication. The cipher used is
AES in CTR mode. A partial implementation is located [here][acp-encryption].

[acp-encryption]: https://github.com/x56/airpyrt-tools/blob/master/acp/encryption.py

This is not fully implemented in the released code. [pysrp][pysrp] differs slightly from AppleSRP which causes the two to
be incompatible. AppleSRP is basically the [SRP reference implementation][srp-ref] modified on the client to use CommonCrypto
and corecrypto rather than OpenSSL.

[pysrp]: https://github.com/x56/airpyrt-tools/blob/master/acp/basebinary.py
[srp-ref]: http://srp.stanford.edu/

In order to experiment with SRP, I [wrote a wrapper][clibs] around the AppleSRP framework using ctypes. This interface 
allows for calling the necessary functions to perform the SRP client or server negotiation, as well as dump several 
structures by operating on pointers returned by these functions for debugging.

[clibs]: https://github.com/x56/airpyrt-tools/blob/master/acp/clibs/AppleSRP.py


## crunchprog Reversing

The userspace binary, crunchprog, contains all of the programs and libraries normally found on a NetBSD system linked
into one massive 10MB+ file (depending on the device and firmware version). With shell access to the device, a list of
included programs can be displayed by creating a hard link to another program named "crunchprog" and executing it.
The statically linked libraries include: libc, libpthread, openssl, CoreFoundation (a modified version) and AppleSRP.

Analyzing this monolithic blob can be tedious. I've [released][cflstring-script] one helpful IDA script to identify static
CFStrings which makes reversing some daemons, including ACPd, a bit easier. One thing to note, the script is fairly hacky
and may need to be run more than once to get all the string references labeled.

[cflstring-script]: https://gist.github.com/x56/8a16c8e30c954aec014d


## NetBSD Toolchain

With shell access to the device, it is possible to push ELF binaries (such as GDB) to assist with further analysis. On 
devices without `scp`, `ssh` and `dd` can be used together to copy over files. Due to the lack of shared libraries on the 
root filesystem, all such programs must be statically linked. Using NetBSD's `build.sh` it's relatively simple to hack 
together a working toolchain for this purpose:

* grab `gnusrc`, `sharesrc`, `src`, and `syssrc` from the [NetBSD archives][NetBSD-ftp]
* extract them all in place
* `cd usr/src/`

First build the cross toolchain (this example is for ARM little endian):

```
./build.sh 5 -U -u -a arm -m evbarm -D ../../evbarm/dest/ -O ../../evbarm/obj/ -R ../../evbarm/release/ -T ../../evbarm/tools/ tools
```

On the second pass, use the same command but swap the final "tools" target with "release" to build all of userspace:

```
./build.sh -U -u -a arm -m evbarm -D ../../evbarm/dest/ -O ../../evbarm/obj/ -R ../../evbarm/release/ -T ../../evbarm/tools/ release
```

Finally, copy all the archives and object files from `evbarm/dest/usr/lib/` to `evbarm/tools/arm--netbsdelf/lib/`.

There is probably a cleaner way to do this but it's what worked for me for building such things as GDB 5. One final note,
due to changes in some kernel/library headers, I found it easiest to spin up an Ubuntu 7.04 VM for a NetBSD 4.0 toolchain
(which is the version running on most AirPort devices).

[NetBSD-ftp]: ftp://ftp.netbsd.org/pub/NetBSD/NetBSD-archive/NetBSD-4.0/source/sets/


## Conclusion

There are quite a few protocol components I didn't yet reverse or implement fully so have left out of this release. The
RPC interface provides access to WPS and DHCP operations as well as other features. Despite the incomplete state of the code
and brevity of this post, I hope it helps others continue to perform research on these devices.

