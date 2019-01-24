---
layout: post
title: Blinking the LED
---

Now that we have a better understanding of how the Raspberry Pi boards boot the kernel, it's time for the Hello World of the bare metal programming: blink the activity LED.

We'll write as much as possible in C99, only using `__asm` statements when absolutely necessary, i.e. to directly access registers. We'll also try to support all the Raspberry Pi boards, so it won't be exactly a walk in the park as we'll see.

For now we'll only deal with AArch32 since there won't be much benefit going with 64-bits now.

Please read [SmartStart32.S explained](https://leiradel.github.io/2019/01/06/SmartStart.html) and [The Raspberry Pi Stubs](https://leiradel.github.io/2019/01/20/Raspberry-Pi-Stubs.html) before moving forward if you haven't already.

## Tooling

We'll use `arm-none-eabi-gcc` and friends to compile bare metal, platform-independent code for our Raspberry Pi. Just install the `gcc-arm-none-eabi` for your system and you'll be able to compile the code presented here.

## The `kmain` Function

As we already know, the kernel will be loaded in memory starting at address `0x00008000`, and this address is also the entry point that will be called by the subs. We also know that this entry point will be called with:

* `r0` set to zero
* `r1` set to the [ARM Linux Machine Type](http://www.arm.linux.org.uk/developer/machines/)
* `r2` set to the address of the [ATAG_CORE](http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html) tag or the [Device Tree](https://elinux.org/Device_Tree_Reference) blob

The [ARM ABI](http://infocenter.arm.com/help/topic/com.arm.doc.subset.swdev.abi/index.html) says that the first four arguments to a C function are passed in registers `r0`, `r1`, `r2`, and `r3`. So our `kmain` function can be written as:

```c
#include <stdint.h>

void kmain(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {
}
```

## The Stack

There are two issues with that though. The first one is that `kmain` is not really a regular C function: we don't have a valid stack so the function cannot set up its function frame. We could write two lines of assembly code starting at `0x00008000` to set up the `sp` register and then jump to `kmain`, but let's try to stay in C land.

`gcc` has a [function attribute](https://gcc.gnu.org/onlinedocs/gcc/ARM-Function-Attributes.html) called `naked`, which won't have the prologue and epilogue sequences as required by the ABI. This means that the compiler won't try to create a function frame and later destroy it, which solves our first issue. That however comes with its own problems: if the compiler decides to save values in the stack inside `kmain`, there won't be any. Even if we set `sp` correctly at the beginning of the function, we won't have a valid function frame unless we setup it manually.

We'll try to stay in C land by doing the absolutely minimum possible inside `kmain`, and checking the compiled code to see if it looks ok.

```c
#include "armregs.h"

#include <stdint.h>

static void __attribute__((noinline)) start(
  const uint32_t machine_type,
  const uint32_t atags_addr) {
}

static uint64_t __attribute__((aligned(8))) stack0[2048];

void __attribute__((naked)) kmain(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {

  set_sp((uint32_t)((uint8_t*)stack0 + sizeof(stack0)));
  start(machine_type, atags_addr);

  while (1) /* nothing */;
}
```

So `kmain` only sets the `sp` register to the end of `stack0`, giving us 16 KB for the stack, calls `start` with the parameters we receive from the stub, and runs an infinite loop to avoid returning to the stub if `start` returns. The compiler will generate a prologue and an epilogue for `start` because it's not marked `naked`, so we'll be able to write any C code we want inside it, but we have to mark it `noinline` to prevent the compiler from inlining it inside `kmain`, which would defeat the purpose.

`stack0` is declared to conform with the ABI, which says that the publicly visible `sp` must be aligned to an eight-byte boundary.

We declare `set_sp` function in `armregs.h` as:

```c
#ifndef ARMREGS_H__
#define ARMREGS_H__

#include <stdint.h>

inline void set_sp(const uint32_t x) {
  __asm volatile("mov sp, %[value]\n" : : [value] "r" (x) : "sp");
}

#endif /* ARMREGS_H__ */
```

## The Entry Point

However, we still have one issue, which is we don't know where the linker will end up putting our code, and we absolutely need `kmain` to start at `0x00008000`. There isn't a compiler attribute that could force `kmain` into going to a specific address, but there's one to force `kmain` into a specific *section*, and we can make the linker put sections at specific locations by writing a [linker script](https://sourceware.org/binutils/docs-2.27/ld/Scripts.html):

```
OUTPUT_ARCH(arm)
ENTRY(kmain)

SECTIONS {
  .text 0x00008000 : {
    KEEP(kmain.o(.kmain))
    *(.text .text.*)
  }
  
  .rodata : {
    . = ALIGN(4);
    *(.rodata .rodata.*)
  }

  .data : {
    . = ALIGN(4);
    *(.data .data.*)
  }

  .bss : {
    . = ALIGN(4);
    *(.bss .bss.*)
    *(COMMON)
  }

  /DISCARD/ : {
    *(*)
  }
}
```

First we set the architecture with `OUTPUT_ARCH(arm)`, and then we set the executable entry point with `ENTRY(kmain)`. I'm not sure those are needed, but there we go. After that, we begin laying out our sections in the `SECTIONS` block in the way we need in order to have `kmain` in the correct address.

We call our first section in the resulting executable `.text`, and we force it to start at `0x00008000`. We then add to this output section the `.kmain` from the `kmain.o` object file, followed by all `.text` and `.text.*` sections from any file. This will make things in the `.kmain` section to be placed at the required address, but only from `kmain.o`.

> Please notice the difference between the function `kmain` and the *section* `.kmain`, with a leading dot.

After `.text` we create an output section `.rodata` for all read-only data. We align their contents at a four-byte boundary with `. = ALIGN(4)` (the dot in the linker script is the [location counter](https://sourceware.org/binutils/docs-2.27/ld/Location-Counter.html), which is similar to the `$` variable in some assemblers).

We repeat the process to add the read-write, initialized data to the `.data` output section, the read-write, uninitialized data to the `.bss`, and then we finish by discarding all other sections, which then won't appear in the final, linked executable.

I'm not sure what goes in the `COMMON` section, but the linker documentation [for this section](https://sourceware.org/binutils/docs-2.27/ld/Input-Section-Common.html#Input-Section-Common) that is usual to add its contents to the `.bss` output section.

With the linker script in place, we can add the section attribute to `kmain`, taking the opportunity to add the `noreturn` attribute since `kmain` never returns:

```c
void __attribute__((section(".kmain"), noreturn, naked)) kmain(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {


  set_sp((uint32_t)(stack0 + sizeof(stack0)));
  start(machine_type, atags_addr);

  while (1) /* nothing */;
}
```

> The compiler won't be able to optimize anything with the `noreturn` attribute since there are no C functions that call `kmain`, but still this is the right thing to do.

## Building

It's easy to build our bare bones program:

```
PREFIX = arm-none-eabi-
CC = $(PREFIX)gcc
LD = $(PREFIX)ld
OBJDUMP = $(PREFIX)objdump
OBJCOPY = $(PREFIX)objcopy
CPPFILT = $(PREFIX)c++filt

CFLAGS   = -fsigned-char -ffreestanding -nostdinc -std=c99 -Os
INCLUDES = -I.
DEFINES  = -DNDEBUG
LDFLAGS  = -T link.T

CFLAGS += -march=armv6k -mtune=arm1176jzf-s -marm -mfpu=vfp -mfloat-abi=hard
TARGET  = kernel

%.o: %.c
	$(CC) $(CFLAGS) $(DEFINES) $(INCLUDES) -c $< -o $@

OBJS = kmain.o

all: $(TARGET).img

$(TARGET).img: $(OBJS)
	$(LD) -o $(TARGET).elf -Map $(TARGET).map $(LDFLAGS) $(OBJS)
	$(OBJDUMP) -d $(TARGET).elf | $(CPPFILT) > $(TARGET).lst
	$(OBJCOPY) $(TARGET).elf -O binary $(TARGET).img
	wc -c $(TARGET).img

clean:
	rm -f $(TARGET).elf $(TARGET).map $(TARGET).lst $(TARGET).img $(OBJS)

.PHONY: clean
```

It's a pretty standard Makefile. The unusual flags are:

* `-fsigned-char`: The `char` type is signed, i.e. `signed char`
* `-ffreestanding`: Signals that the standard library my not exist, that the program entry point may not be `main`, and remove support for the builtin functions
* `-nostdinc`: Prevents the preprocessor from searching the default system header directories for included files
* `-march=armv6k`: Sets the target architecture
* `-mtune=arm1176jzf-s`: Tune the generated code for this CPU
* `-marm`: Generates ARM instructions instead of Thumb
* `-mfpu=vfp`: Specifies the floating-point hardware
* `-mfloat-abi=hard`: Specified that the floating-point ABI uses hardware registers for function calls

> The compiler flags for C dialects are documented [here](https://gcc.gnu.org/onlinedocs/gcc/C-Dialect-Options.html), and the ARM-specific ones [here](https://gcc.gnu.org/onlinedocs/gcc/ARM-Options.html). The preprocessor `-nostdinc` is documented [here](https://gcc.gnu.org/onlinedocs/cpp/Search-Path.html).

We use `-ffreestanding` and `-nostdinc` to have a completely blank state in terms of standard libraries and headers. The reason is to make sure we only add to our bare bones program what we absolutely need or want.

We need `-fsigned-char` to force a signed `char` type because, quite surprisingly to me, `arm-none-eabi-gcc` defaults to an unsigned `char` type:

```
$ arm-none-eabi-gcc -dM -E -x c /dev/null |grep CHAR_UNSIGNED
#define __CHAR_UNSIGNED__ 1
```

Since we're leaving out the standard headers, we have to cherry-pick the ones that we want to use. It's just a matter of copying these files from the `gcc-arm-none-eabi` package. `stdint.h`, `stdint-gcc.h`, and `stddef.h` will do for now.

It's also interesting to understand how the final `kernel.img` image is created:

1. `$(LD) -o $(TARGET).elf -Map $(TARGET).map $(LDFLAGS) $(OBJS)`: Creates the ELF executable, and a map file with the detailed content of the sections in the ELF
1. `$(OBJDUMP) -d $(TARGET).elf | $(CPPFILT) > $(TARGET).lst`: Creates a listing of the executable with demangled symbols (if we ever add C++ code to it)
1. `$(OBJCOPY) $(TARGET).elf -O binary $(TARGET).img`: Creates a binary file with the code and data, ready to be loaded and run
1. `wc -c $(TARGET).img`: Counts the final executable size, so we keep an eye on it

## The Listing File

For our current kernel, this is the listing file created by the Makefile:

```
kernel.elf:     file format elf32-littlearm


Disassembly of section .text:

00008000 <kmain>:
    8000:	e59f3004 	ldr	r3, [pc, #4]	; 800c <kmain+0xc>
    8004:	e1a0d003 	mov	sp, r3
    8008:	eafffffe 	b	8008 <kmain+0x8>
    800c:	0000c010 	.word	0x0000c010
```

For now our program looks good: the compiler didn't generate a function frame, and is setting the stack pointer register at the end of the code plus 16 KB.

With these details behind us, we will be able to write the `start` function in the next post.
