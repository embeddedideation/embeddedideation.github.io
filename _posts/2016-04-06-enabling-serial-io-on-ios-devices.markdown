---
layout:    post
title:     "Enabling Serial I/O on iOS Devices"
date:      2016-04-06 02:45
---

This post describes a technique for enabling the serial console on iOS devices up to iOS 9. It may work on iOS 9 devices
as well, but I haven't tested this and KPP may or may not be an issue. This was loosely based off of [a method][serialplease]
used by comex, but described in more detail. It should work on 32- and 64-bit devices, with either a 30-pin or Lightning 
connector. Construction of a serial cable is beyond the scope of this post.

[serialplease]: https://github.com/comex/white/blob/master/serialplease.c

I [tweeted][tweet-1] [about][tweet-2] [this][tweet-3] previously, but did not go into much detail at the time. I was asked
about this again recently so decided to consolidate the information and elaborate so it would be easier to find the necessary
kernel functions and globals to perform this on an arbitrary firmware.

[tweet-1]: https://twitter.com/0x56/status/620442820432695296
[tweet-2]: https://twitter.com/0x56/status/620442932881928192
[tweet-3]: https://twitter.com/0x56/status/620483484117700608

Note that you can, if you have a way to modify boot-args before kernel initialization, make your life easier by simply 
adding `debug=0x8 serial=3`.


## Overview

There are a couple of things that must be done in order to activate the serial I/O fully: initializing the UART 
hardware and starting a kernel thread to poll for input. KDP uses the PAL serial routines (described below) directly 
and should work after these steps. Optionally, a console can be enabled to send debug output over serial as well.


### Initializing UART

During platform initialization, various boot arguments are parsed to enable or disable a variety of kernel features. One
well-known argument, `debug`, takes an argument which comprises a series of bit flags (defined in `osfmk/kern/debug.h`)
ORed together. The `DB_KPRT` flag determines if serial output should be enabled, and if so initializes the hardware.

Please note that, although the following code is for i386 devices, the implementation for ARM devices is very similar.

~~~
void PE_init_kprintf(boolean_t vm_initialized)
{
	[...]
	
	if (PE_parse_boot_argn("debug", &boot_arg, sizeof (boot_arg)))
		if (boot_arg & DB_KPRT)
			new_disable_serial_output = FALSE;
	
	[...]
	
	if (!new_disable_serial_output && (!disable_serial_output || pal_serial_init()))
		PE_kputc = pal_serial_putc;
	else
		PE_kputc = cnputc;
	
	disable_serial_output = new_disable_serial_output;
	
	[...]
}	
~~~
_From pexpert/i386/pe_kprintf.c_

The PAL (Platform Abstraction Layer) serial routines (found in `osfmk/i386/pal_routines.c`) simply dispatch to 
their implementations in the platform expert (found in `pexpert/i386/pe_serial.c`). `pal_serial_init()` calls
`serial_init()` which actually does the work of initializing the UART. Luckily for us, this function still has a symbol 
(at least through iOS 8) and is thus easy to locate by parsing `SYMTAB`. Once located, simply call the function 
(which takes no arguments) to start the hardware.


### Selecting the Console Mode

Shortly after the UART hardware is up and running, the `serial` boot argument is parsed in order to determine the value
of `serialmode`. This is a kernel global variable comprising two flags:

~~~
(1 << 0): enable serial output
(1 << 1): enable serial input
~~~

If console output is desired, the following happens:

~~~
void
i386_init(void)
{
	[...]
	
	if(serialmode & 1) {
		(void)switch_to_serial_console();
		disableConsoleOutput = FALSE;	/* Allow printfs to happen */
	}
	
	[...]
}
~~~
_From osfmk/i386/i386_init.c_

`switch_to_serial_console()` stores the current console settings and sets `cons_ops_index` for serial I/O.

~~~
int
switch_to_serial_console(void)
{
	int old_cons_ops = cons_ops_index;
	cons_ops_index = SERIAL_CONS_OPS;
	return old_cons_ops;
}
~~~
_From osfmk/console/serial_general.c_

The `console_ops` structure and valid indices for the `cons_ops` array are defined in `osfmk/console/serial_protos.h`:

~~~
struct console_ops {
	void (*putc)(int, int, int);
	int  (*getc)(int, int, boolean_t, boolean_t);
};

#define SERIAL_CONS_OPS 0
#define VC_CONS_OPS 1
~~~

The `cons_ops` array itself is defined in `osfmk/console/i386/serial_console.c`:

~~~
struct console_ops cons_ops[] = {
	{
		.putc = _serial_putc,
		.getc = _serial_getc,
	},
	{
		.putc = vcputc,
		.getc = vcgetc,
	},
};
~~~


### Ungating Console Output

By default on a production kernel, debug output is only sent to `syslog`. To also print this information on the console,
a couple of global variables must be adjusted.

`printf()` and `IOLog()` are two functions used throughout the kernel for logging various interesting things. `printf()`
calls `conslog_putc()` to handle writing it's output to the console. Similarly, `IOLog()` calls `cons_putc_locked()`.
Both functions contain the following line:

~~~
if ((debug_mode && !disable_debug_output) || !disableConsoleOutput)
	cnputc(c);
~~~
_From osfmk/kern/printf.c_

Simply, if `disableConsoleOutput` is `FALSE`, pass the character to `cnputc()`.

`cnputc()` dispatches the `putc()` call to the appropriate handler based on the setting of `cons_ops_index`:

~~~
static inline void
_cnputc(char c)
{
	[...]
	
	cons_ops[cons_ops_index].putc(0, 0, c);
	if (c == '\n')
		cons_ops[cons_ops_index].putc(0, 0, '\r');

	[...]
}
~~~
_From osfmk/console/i386/serial_console.c_

`kprintf()` also calls through to `cnputc()` by default in production kernels, but is gated by a separate global variable.
`disable_serial_output` is set during `PE_init_kprintf()` based on the value of `DB_KPRT`. If this is `FALSE`, the body of
`kprintf()` is skipped:

~~~
void kprintf(const char *fmt, ...)
{
	[...]

	if (!disable_serial_output) {
		
		[... printing stuff ...]
	}
}
~~~
_From pexpert/i386/pe_kprintf.c_

Therefore, by inverting the default values of `disableConsoleOutput` and `disable_serial_output` all debug printing should
happen over the console as well.

One thing to note, on the iOS kernels I have analyzed the boolean `disable_serial_output` seems to be reversed. That is, 
it would be more accurately named `enable_serial_output`. I am unsure if this is due to the compiler or if the ARM 
implementation differs here from the i386 one. Regardless, inverting this value should enable `kprintf()` output.


### Enabling the Serial Keyboard

A while later in kernel bringup, during `kernel_bootstrap_thread()` (found in `osfmk/kern/startup.c`), 
`serial_keyboard_init()` is called. This function performs the following:

~~~
void
serial_keyboard_init(void)
{
	[...]

	if(!(serialmode & 2)) /* Leave if we do not want a serial console */
		return;

	kprintf("Serial keyboard started\n");
	result = kernel_thread_start_priority((thread_continue_t)serial_keyboard_start, NULL, MAXPRI_KERNEL, &thread);
	if (result != KERN_SUCCESS)
		panic("serial_keyboard_init");

	thread_deallocate(thread);
}
~~~
_From osfmk/console/serial_general.c_

This function seems to be inlined in iOS kernels and `serialmode` is set to `0` by default. Given these circumstances,
I found it easier to write a [small function][cereal64] that can be run from kernelmode to start the polling thread 
manually. This function is written in A64 but will work if ported to A32.

[cereal64]: https://gist.github.com/x56/4f2dbf2f5b267d939d84


## Locating Kernel Functions and Globals

Several functions and global variables must be located to perform this method. `serial_init()` and `thread_deallocate()`
have symbols available up through iOS 8 kernels and can be looked up from `SYMTAB`. The other necessary components
require a bit more effort to find.

Even though `serial_keyboard_init()` can be inlined, the function body is marked by the call to `panic()` with an argument
of the function's name. Comparing this to the open source code, locating the necessary functions should be trivial.

A pointer to `serial_keyboard_start()` is passed to `kernel_thread_start_priority()` just prior to the aforementioned
call to `panic()`. `serial_keyboard_start()` similarly calls `panic()` with it's name as part of the string passed.

`kernel_thread_start_priority()` takes the `serial_keyboard_start()` pointer and 0x5F (the value of `MAXPRI_KERNEL`) as 
arguments.

`disableConsoleOutput` is referenced in `conslog_putc()` (which has a symbol) directly before the call to `cnputc()`.

`cons_ops_index` is referenced in `cnputc()` right before the `cons_ops` array (which also has a symbol).

`disable_serial_output` is the first boolean referenced in `kprintf()` (which again has a symbol) and determines if the
rest of the function should be run.

Tools such as [patchfinder][patchfinder] exist to find functions and variables based on common instruction patterns and
string references. Given the descriptions above, generalizing this process for all firmwares shouldn't be too difficult.

[patchfinder]: https://github.com/planetbeing/ios-jailbreak-patchfinder/blob/master/patchfinder.c


## Summary

* call `serial_init()`
* run `serial_keyboard_start()` in an appropriate kernel thread for input
* set `cons_ops_index` to `0`
* invert `disableConsoleOutput` for `printf()`, `IOLog()`, etc. output
* invert `disable_serial_output` for `kprintf()` output

I've tested this process on multiple 64-bit devices, but as we're simply following the same initialization process used in
XNU (albeit at a later point) this should be universal.

As always, [feedback is welcome][contact] if I've missed something. I'd also be interested to hear if this works on iOS 9
if anyone gets around to trying.

[contact]: http://embeddedideation.com/about/
