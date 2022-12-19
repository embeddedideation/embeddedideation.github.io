---
layout:    post
title:     "Exploring the Physical Address Space on iOS"
date:      2016-03-10 01:03
---

Interacting directly with platform hardware can be very useful. Doing so can circumvent security features, and allows one 
to glean more information about the underlying system. In the early days of iOS neither the software nor the platform were 
well secured leading to many techniques for dumping firmwares, reverse engineering, and developing custom software for these 
devices. Boot ROM vulnerabilities in particular allowed for controlling devices shortly after they were powered on. After 
all such public vulnerabilities were patched, techniques such as kloader leveraged system design to allow for interacting 
with the device in a state conducive to exploration. With the release of ARMv8 based devices utilizing the secure world, 
the situation became more complicated still, as there are no public vulnerabilities that allow for running software on the 
device at early stages of system bringup or directly dumping protected firmware. There are however a number of kernel and 
architecture functionalities that can be exploited to ease digging into the platform and extracting useful information, 
some of which will be discussed in this post.


## Prerequisites

From userspace, you are primarily limited to only your own virtual address space, and calling BSD syscalls or Mach/IOKit 
traps to run kernel code. So to go deeper you must first have the ability to run code in the kernel context. Jailbreaks 
traditionally patch the kernel to allow returning a task port for PID 0 to read and write kernel memory. With the 
ability to read/write arbitrary kernel memory you can readily allocate a page, write in a payload, hook a `sy_call` or
IOKit object vtable, and call a trap to run your payload in kernelmode. Since iOS 9, Kernel Patch Protection makes it
infeasible to patch `task_for_pid()` persistently so you must find other ways to get the kernel task port or otherwise 
access kernelspace. Reversing the untether binary on your device and reusing the kernel exploit to bootstrap your payload 
is certainly an option, though not ideal for various reasons including heavy code obfuscation. Regardless of your chosen 
method this is outside the scope of this post. From here on, it is assumed you have the ability to access kernelspace and 
achieve code execution.


## Accessing Physical Address Space

Many interesting things are accessible somewhere on the system bus. Since the MMU is enabled during normal operation 
of the device, physical addresses cannot be accessed without first mapping them into some task's address space. The 
kernel maps DRAM addresses in it's own address space through memory allocators, or in usermode task address spaces
during process bringup. Hardware drivers create mappings to specific register regions in order to configure, control, 
and query the status of peripherals.

There are a few ways to do this with kernel code. Perhaps most directly, you can locate a task's translation tables
and manually write entries. This technique is described in detail in the ARM Architecture Reference Manuals, as
it is the low level method for setting up address translation. This was used by past jailbreaks after sysent was made 
read only in order to facilitate syscall hooking for a userspace to kernelspace trampoline. This is also used by winocm's 
shadowmap technique to allow reading and writing kernel memory from userspace. Ultimately, the existing kernel APIs for 
memory mapping do this as well. In XNU, two components are primarily responsible for memory interactions: Mach and IOKit. 


### ml_io_map()

In the Mach layer, `ml_io_map()` is used during kernel boot to create mappings to physical regions corresponding 
to device tree entries needed in the platform expert. This function takes a physical address and size parameter, then 
calls through to `io_map()` additionally specifying the `VM_MEM_GUARDED`, `VM_MEM_COHERENT`, and `VM_MEM_NOT_CACHEABLE` flags. 
These flags setup the mapped region as device memory, meaning the cache will not interfere with future reading or writing. 
Entries are then added to `kernel_map` via `pmap_map()` and a kernel virtual address pointing to the start of the region is 
returned.

If freeing this mapping later is desired, `kmem_free()` can be called with `kernel_map`.

Usage example:

~~~
extern vm_map_t kernel_map;

vm_offset_t t7001GPIOBase = 0x20e300000;
vm_size_t   t7001GPIOSize = 0x100000;

vm_offset_t t7001GPIORegion = ml_io_map(t7001GPIOBase, t7001GPIOSize);

/* do stuff */

kmem_free(kernel_map, t7001GPIORegion, t7001GPIOSize);
~~~


### IOMemoryDescriptor (and family)

IOKit provides several classes for drivers to map memory in various ways. These classes derive from IOMemoryDescriptor
(or from its subclass IOGeneralMemoryDescriptor, used internally by the parent) and are useful for varying purposes.

For our needs, **IOMemoryDescriptor** is the most helpful class. `IOMemoryDescriptor::withPhysicalAddress()` takes
a physical address, size, and direction parameter and returns an IOMemoryDescriptor object. This descriptor
contains a reference to an IOMemoryMap object which is detailed in a moment. Another very nice trick is accomplished
with `IOMemoryDescriptor::createMappingInTask()`. By calling this member of the descriptor, you can get a new mapping
of the physical region inside an arbitrary task. By passing the task port of a user process as the first parameter
the given task will have direct access to the region, so further experimentation can be performed directly from userspace.

**IODeviceMemory** provides convenience functions for creating descriptors to regions of physical address space.
`IODeviceMemory::withRange()` is a wrapper around `IOMemoryDescriptor::withAddressRange()`, however the default options
passed may not be desirable for your use case.

Another possibly useful class is **IOBufferMemoryDescriptor**. This is similar to IOMemoryDescriptor with the addition
of allocating a buffer pointed to by the descriptor's map. This is not so useful for interacting with hardware but
can give you, for example, a chunk of contiguous physical memory with caching disabled that can be mapped into
a userspace task that will be left alone by other tasks (quite valuable for some experiments).

In order to make use of an IOMemoryDescriptor object directly, it must be mapped into a task via either 
`IOMemoryDescriptor::map()` (into the kernel map) or `IOMemoryDescriptor::createMappingInTask()` (any specified task's
map). These methods return an **IOMemoryMap** object. This object's `getVirtualAddress()` and `getPhysicalAddress()` 
methods are then used to determine the location of the mapped region.

Usage example:

(NOTE: I implemented this originally in A64 but provide C++ here for brevity. This hasn't been tested but Should Work™.)

~~~
IOPhysicalAddress t7000SRAMBase = 0x180000000;
IOByteCount       t7000SRAMLength = 0x200000;

IOMemoryDescriptor *descriptor = IOMemoryDescriptor::withPhysicalAddress(t7000SRAMBase,
                                                                         t7000SRAMLength,
                                                                         kIODirectionOutIn);

/* can lookup in userspace with task_for_pid() and pass in */
task_t task = port_name_to_task(user_task_port);

IOMemoryMap *map = descriptor->createMappingInTask(task, 0, kIOMapAnywhere, 0, 0);

IOVirtualAddress userAddress = map->getVirtualAddress();
~~~


For full documentation of these classes, consult each [class reference][mac-developer-library] and the 
[XNU source][xnu-source].

[mac-developer-library]: https://developer.apple.com/library/mac/navigation/
[xnu-source]: https://opensource.apple.com/source/xnu/


## Interesting Targets

A variety of things are possible with direct access to the physical address space. The physical DRAM base can be 
easily determined by inspecting TTEs or reading the value of `gPhysBase`. The location of peripherals can be more elusive 
however. It is possible to query the IORegistry to retrieve the IODeviceMemory property of loaded kexts to find physical 
regions associated with peripherals. Further reversing of these kexts can indicate what the function of specific registers 
are inside those regions. Some other peripherals are unused by XNU, but their configuration registers are still accessible. 
They can be located by reversing iBoot or the secure world kernel/monitor, or lacking those firmwares, by brute force 
searching of the physical address space (this is not trivial on 64-bit devices but in some cases educated guesses are 
possible). There are likely other interesting targets, however attempts to access them from kernelspace causes strange 
behavior (such as a hang and reboot with no panic log) possibly due to incorrect methodology or bus protection mechanisms.


### DRAM

With direct access to system RAM, dumping and patching various bits of code becomes much simpler. You must translate
virtual to remapped addresses, but once the offsets are determined calculations are trivial.


#### Kernel Patching

When working with kernel memory, since iOS 6 the kernel's virtual base is randomized between one of 256 possible locations
on boot. The base in physical memory is not however, and is only determined by the device model and firmware version. On 
32-bit devices this is either `0x40000000` (A4 and earlier) or `0x80000000` (A5 and newer). With 64-bit devices, DRAM is 
based at `0x800000000` however the kernel base is located after a region of several MB reserved for the secure world. For 
example, with the iPad5,4 on iOS 8.1 XNU starts at `0x800e00000`.

One thing to note, when reading or writing directly to RAM the cache is bypassed so the data that is read could conceivably
by stale, and any changes (such as patching kernel text) will require invalidating the relevant cache (in that case 
instruction cache).


#### Firmware Recovery

DRAM is shared in multiple contexts and not all of them are fully isolated from one another. During boot, LLB is loaded
to SRAM but iBoot is loaded to DRAM. Prior to iOS 9 at least, iBoot was not cleared from memory before jumping to the 
kernel so by mapping and scanning DRAM it is possible to recover the firmware.

On 64-bit devices, DRAM is used by other coprocessors as well. The Apple Storage Processor (NAND controller) has its
firmware loaded by iBoot into the uppermost several MB of DRAM. This is another 32-bit ARM firmware based on iBoot. Since
it is located in a region of memory reachable via the AP, it can be dumped and possibly altered while running (though
not persistently as it is contained in the iBoot image, which is signature checked on each boot).


### Hardware Registers

Any SoC uses special regions of physical addresses to interact with peripherals. Apple's chips are no exception, although
the reference manual identifying the use of these regions is certainly not public. If these regions can be located
you can still read and write to them to control device hardware. GPIOs can be read/written directly, interrupts can be
masked and cleared, clocks can be configured; anything drivers can do you can do also. Some very interesting registers, such 
as those used to control the TZASC appear to be locked or one-shot, so can't be altered from kernelmode.

It should be noted that since this is not a bare SoC on a test jig, mucking around with hardware directly is likely to
cause errors, by interfering with ongoing OS operations or putting the board into an unexpected state. If things go wrong,
hopefully a mechanism such as a watchdog timer will kick the device into reset, but this is not guaranteed. You may need 
to forcefully reboot the device by holding the power and home buttons, or even open the device to disconnect and reconnect
the battery. In the worst case, permanent hardware damage is possible.


### Boot ROM

On some 32-bit devices (S5L892x and S5L8930), the mask ROM containing the SecureROM bootloader is available on the bus 
as a memory peripheral. By mapping a 64kB region at `0xbf000000` and reading, this firmware can be retrieved 
directly.


### SRAM/Cache

In more exotic and inexplicable finds, there is a 4MB range located at `0x180000000` on 64-bit devices that 
seems to have multiple uses. Based on serial console output during boot it corresponds with the SRAM region used by 
LLB, but reading this from XNU does not get you this firmware. Instead it seems to be split into 2 regions of equal size.
On T7000 devices, reading the first 2MB region gets you rapidly changing data. After some experimentation, it became clear 
this was some sort of cache, as it contained 64 byte chunks of data from running programs (though sadly it did not seem to 
contain anything obviously from the secure world or other coprocessors). The second half of this region only caused a brief 
hang followed by a reboot, however not the typical panic caused by the memory controller attempting to access a memory hole. 
On T7001 devices, accesses to this entire region only seem to trigger a hang and reboot and do not yield cache data.


## Caveats

As previously mentioned, **permanent damage is possible when playing directly with hardware**. For example, OTP fuses are
typically set via a particular sequence of register accesses and inadvertently performing this through fuzzing is 
theoretically possible. Changing GPIO configuration could lead to excessive amounts of current sinking into a port not 
meant to handle this. Signals sent to other devices on the board could cause them to enter a problematic state. Without 
a reference manual, you can't be sure what effects your actions will have, so be careful.

Accesses to alternate mappings of DRAM can be performed without special considerations, however those to peripherals
typically must be of a specific width. On 64-bit devices, this usually means using 32-bit accesses at a 64-bit offset:

~~~
/* 32-bit read */
LDR w1, [x0]

/* 32-bit write */
STR w1, [x0]
~~~

Failure to do this results in a kernel panic: possibly an unaligned kernel data abort or a fault triggered by the memory
controller.


## Conclusion

A variety of kernel functions exist that can be harnessed to ease interaction with the underlying platform. Even without
access to these functions, running code in kernelmode to leverage architectural features allows accomplishing the same
ends, though it requires more work and understanding of these features. These tricks can be valuable to someone looking
to learn more about a given platform, or to dig into undocumented realms and exploit unexpected and undefined states of the
system. It is also important for those auditing a new platform to be aware of these techniques so devices can be hardened
and access restricted to areas that should not be exposed, even to the kernel.
