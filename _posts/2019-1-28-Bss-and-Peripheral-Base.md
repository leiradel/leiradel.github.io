---
layout: post
title: Blinking the LED Part 2 - Clearing the .bss and the Peripherals Base Address
---

## The `start` Function

When we write a regular C program, the C runtime takes care of a number of things for us. One of those things is clearing the `.bss` section, which is where the uninitialized, statically allocated variables reside, as well as statically allocated variables that are initialized to values represented by all bits being 0.

The operating system or the C runtime make sure that this section is filled with zeroes before execution reaches `main`. Since we don't have that kind of help, we have to zero the `.bss` section ourselves.

> Our program only worked until now because `stack0`, which is where the stack is created, doesn't require any kind of initialization.

To zero `.bss`, we need to know where in memory it starts and ends. Fortunately, the linker script can create symbols with that information for us:

```
  .bss : {
    . = ALIGN(4);
    bss_start = .;
    *(.bss .bss.*)
    *(COMMON)
    . = ALIGN(4);
    bss_end = .;
  }
```

We can now use `bss_start` and `bss_end` in C code to zero the `.bss` section, which is the first thing we will do in `start` to make sure everything relying on it works:

```c
static void __attribute__((noinline)) start(
  const uint32_t machine_type,
  const uint32_t atags_addr) {

  // Zero the .bss section.
  extern uint32_t bss_start, bss_end;

  for (uint32_t* i = &bss_start; i < &bss_end; i++) {
    *i = 0;
  }
}
```

The linker creates symbols as if they were variables at the locations specified in the linker script, but without any associated storage. This can be checked by looking at the output of `objdump` (I've removed most of the output for brevity):

```
$ arm-none-eabi-objdump -x kernel.elf

SYMBOL TABLE:
0000c040 g       .bss	00000000 bss_end
00008040 g       .bss	00000000 bss_start
```

The zeros right before the symbols' names are their sizes, which are zero.

The "correct" way to declared them in C would be `extern void bss_start, bss_end;`, and `i` would be `void*`, but then the compiler warns us about taking the address of expressions of type `void`. To *avoid* these warnings, and also the casts needed to write to `*i` and update it, we declare them as `uint32_t`. This does *not* allocate space for them, if we use their values we'll see values from whatever is in there.

> I couldn't find any reference for this, the explanation is based purely on tests and common sense.

The choice of `uint32_t` is not random, it has the benefit of letting us declare `i` as `uint32_t*` and write four aligned bytes in one go. That's why we align the location counter to a four-byte boundary before declaring `bss_start` and `bss_end` in the linker script.

If we compile now, we'll be able to check in `kernel.lst` that `.bss` is indeed being cleared:

```
00008000 <kmain>:
    8000:	e59f3008 	ldr	r3, [pc, #8]	; 8010 <kmain+0x10>
    8004:	e1a0d003 	mov	sp, r3
    8008:	eb000001 	bl	8014 <start.isra.0>
    800c:	eafffffe 	b	800c <kmain+0xc>
    8010:	0000c040 	.word	0x0000c040

00008014 <start.isra.0>:
    8014:	e3a01000 	mov	r1, #0
    8018:	e59f3014 	ldr	r3, [pc, #20]	; 8034 <start.isra.0+0x20>
    801c:	e59f2014 	ldr	r2, [pc, #20]	; 8038 <start.isra.0+0x24>
    8020:	e1530002 	cmp	r3, r2
    8024:	3a000000 	bcc	802c <start.isra.0+0x18>
    8028:	e12fff1e 	bx	lr
    802c:	e4831004 	str	r1, [r3], #4
    8030:	eafffffa 	b	8020 <start.isra.0+0xc>
    8034:	00008040 	.word	0x00008040
    8038:	0000c040 	.word	0x0000c040
```

> The funny function name `start.isra.0` is related to compiler optimizations that take place with `-Os`, as explained in this [Stack Overflow answer](https://stackoverflow.com/questions/18907580/what-is-isra-in-the-kernel-thread-dump).

## Small Optimization

Now that `start` has something in it and the compiler generates code to call it, lets see something interesting from the ABI.

Lets change the loop that clears the `.bss` section to use the two parameters passed from `kmain`:

```c
  for (uint32_t i = machine_type; i < atags_addr; i += 4) {
    *(uint32_t*)i = 0;
  }
```

> This code doesn't do anything useful nor correct, it's there just to make sure the compiler uses the parameters passed from `kmain`.

If we look to `kmain` in the listing, we'll see it moving values between registers before calling `start`:

```
00008000 <kmain>:
    8000:	e1a00001 	mov	r0, r1
    8004:	e59f300c 	ldr	r3, [pc, #12]	; 8018 <kmain+0x18>
    8008:	e1a0d003 	mov	sp, r3
    800c:	e1a01002 	mov	r1, r2
    8010:	eb000001 	bl	801c <start>
    8014:	eafffffe 	b	8014 <kmain+0x14>
    8018:	0000c038 	.word	0x0000c038
```

`kmain` receives `zero`, `machine_type`, and `atags_addr` in `r0`, `r1`, and `r2` from the stub. However, `start` only wants `machine_type` and `atags_addr`, which then need to go into `r0` and `r1`. The compiler must move `r1` to `r0` and `r2` to `r1` to put the values in the correct registers before calling main. Maybe we can avoid this by declaring `start` to also receive `zero`?

```c
static void __attribute__((noinline)) start(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {

  ...
}
```

Unfortunately (or fortunately!), the compiler sees that `zero` is not being used in `start`, and that `start` is only being called from `kmain`, and optimizes away this parameter as if it doesn't exist. Lets use it in our loop:

```c
  for (uint32_t i = machine_type; i < atags_addr; i += 4) {
    *(uint32_t*)i = zero;
  }
```

We can see in the listing that the compiler was able to greatly reduce the generated code:

```
00008000 <kmain>:
    8000:	e59f3008 	ldr	r3, [pc, #8]	; 8010 <kmain+0x10>
    8004:	e1a0d003 	mov	sp, r3
    8008:	eb000001 	bl	8014 <start>
    800c:	eafffffe 	b	800c <kmain+0xc>
    8010:	0000c028 	.word	0x0000c028

00008014 <start>:
    8014:	e1510002 	cmp	r1, r2
    8018:	3a000000 	bcc	8020 <start+0xc>
    801c:	e12fff1e 	bx	lr
    8020:	e4810004 	str	r0, [r1], #4
    8024:	eafffffa 	b	8014 <start>
```

Not only values in registers don't need to be moved around in `kmain`, but the compiler didn't need to initialize a register to 0 since `r0` is being used to clear the `.bss`. Lets fix the code to use `bss_start` and `bss_end`:

```c
  for (uint32_t* i = &bss_start; i < &bss_end; i++) {
    *i = zero;
  }
```

This compiles to:

```
00008000 <kmain>:
    8000:	e59f3008 	ldr	r3, [pc, #8]	; 8010 <kmain+0x10>
    8004:	e1a0d003 	mov	sp, r3
    8008:	eb000001 	bl	8014 <start.isra.0>
    800c:	eafffffe 	b	800c <kmain+0xc>
    8010:	0000c038 	.word	0x0000c038

00008014 <start.isra.0>:
    8014:	e59f3014 	ldr	r3, [pc, #20]	; 8030 <start.isra.0+0x1c>
    8018:	e59f2014 	ldr	r2, [pc, #20]	; 8034 <start.isra.0+0x20>
    801c:	e1530002 	cmp	r3, r2
    8020:	3a000000 	bcc	8028 <start.isra.0+0x14>
    8024:	e12fff1e 	bx	lr
    8028:	e4830004 	str	r0, [r3], #4
    802c:	eafffffa 	b	801c <start.isra.0+0x8>
    8030:	00008038 	.word	0x00008038
    8034:	0000c038 	.word	0x0000c038
```

> I know this is an example of early and micro-optimization, but I can't help myself.

## Blinking the LED

The activity LED on the Raspberry Pi boards is connected to a [general-purpose input/output](https://en.wikipedia.org/wiki/General-purpose_input/output) (GPIO) pin. All it's needed to turn it on and off is to program its pin as an output pin, and set the pin level to high and low.

The GPIO is a memory-mapped peripheral, so we need to know both its memory location and its memory layout to be able to program the pins and set their level. The GPIO memory layout is documented in [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), chapter 6 - General Purpose I/O (GPIO).

The memory location is physical address `0x20000000`, but this document is for boards based on the BCM2835 SoC and the address has changed to physical address `0x3e000000` on boards based on later SoCs, as documented in [QA7 - ARM Quad A7 core](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2836/QA7_rev3.4.pdf). We have to find a way to discover the correct address at runtime, but lets tackle the memory layout first.

## GPIO Memory Layout

The most common way use a memory-mapped peripheral is to create a structure with the exact same layout as the peripheral, and then declare a variable pointing to a structure of that type at the memory base of the peripheral. The Raspberry Pi GPIO could be declared as:

```c
typedef struct __attribute__((packed)) {
  uint32_t GPFSEL0;    // 0x00
  uint32_t GPFSEL1;    // 0x04
  uint32_t GPFSEL2;    // 0x08
  uint32_t GPFSEL3;    // 0x0c
  uint32_t GPFSEL4;    // 0x10
  uint32_t GPFSEL5;    // 0x14
  uint32_t reserved1;  // 0x18
  uint32_t GPSET0;     // 0x1c
  uint32_t GPSET1;     // 0x20
  uint32_t reserved2;  // 0x24
  uint32_t GPCLR0;     // 0x28
  uint32_t GPCLR1;     // 0x2c
  uint32_t reserved3;  // 0x30
  uint32_t GPLEV0;     // 0x34
  uint32_t GPLEV1;     // 0x38
  uint32_t reserved4;  // 0x3c
  uint32_t GPEDS0;     // 0x40
  uint32_t GPEDS1;     // 0x44
  uint32_t reserved5;  // 0x48
  uint32_t GPREN0;     // 0x4c
  uint32_t GPREN1;     // 0x50
  uint32_t reserved6;  // 0x54
  uint32_t GPFEN0;     // 0x58
  uint32_t GPFEN1;     // 0x5c
  uint32_t reserved7;  // 0x60
  uint32_t GPHEN0;     // 0x64
  uint32_t GPHEN1;     // 0x68
  uint32_t reserved8;  // 0x6c
  uint32_t GPLEN0;     // 0x70
  uint32_t GPLEN1;     // 0x74
  uint32_t reserved9;  // 0x78
  uint32_t GPAREN0;    // 0x7c
  uint32_t GPAREN1;    // 0x80
  uint32_t reserved10; // 0x84
  uint32_t GPAFEN0;    // 0x88
  uint32_t GPAFEN1;    // 0x8c
  uint32_t reserved11; // 0x90
  uint32_t GPPUD;      // 0x94
  uint32_t GPPUDCLK0;  // 0x98
  uint32_t GPPUDCLK1;  // 0x9c
}
gpio_t;

volatile gpio_t* get_gpio(void) {
  // Return a pointer to the GPIO base.
}
```

This is very reasonable and would be my choice to use memory-mapped peripherals, but lets take a look at this piece of code:

```c
void wait(void) {
  volatile gpio_t* gpio = get_gpio();
  uint32_t gppud;

  do {
    gppud = gpio->GPPUD;
  }
  while ((gppud & UINT32_C(0x40000000)) != 0);
}
```

> `GPPUD` is not intended to be used this way, this code is just to illustrate the issue.

Using the compilation flags in the Makefile, we get this code:

```
00000000 <test>:
   0:	e92d4010 	push	{r4, lr}
   4:	ebfffffe 	bl	0 <get_gpio>
   8:	e5d03094 	ldrb	r3, [r0, #148]	; 0x94
   c:	e5d03095 	ldrb	r3, [r0, #149]	; 0x95
  10:	e5d03096 	ldrb	r3, [r0, #150]	; 0x96
  14:	e5d03097 	ldrb	r3, [r0, #151]	; 0x97
  18:	e3130040 	tst	r3, #64	; 0x40
  1c:	1afffff9 	bne	8 <test+0x8>
  20:	e8bd8010 	pop	{r4, pc}
```

I won't even try to understand why the compiler has done this, but what matters is that, instead of reading four bytes in one go, the code reads the least-significant byte first, then the second LBS, the third, and then the MSB, which is then tested against `0x40`.

While this works for addresses that have memory semantics, here we're dealing with a peripheral. Many peripherals have collateral effects when read and/or written, i.e. reading from a peripheral's register may clear it's contents. If that happens with `GPPUD`, the loop condition could miss the test value and spin forever.

To force atomically reading and writing to peripheral registers that are four bytes wide, we'll write two more little assembly functions:

```c
#ifndef MEMIO_H__
#define MEMIO_H__

#include <stdint.h>

inline uint32_t mem_read32(const uint32_t address) {
  uint32_t result;

  __asm volatile(
    "ldr %[result], [%[address]]\n"
    : [result] "=r" (result)
    : [address] "r" (address)
  );

  return result;
}

inline void mem_write32(const uint32_t address, const uint32_t value) {
  __asm volatile(
    "str %[value], [%[address]]\n"
    :
    : [address] "r" (address), [value] "r" (value)
  );
}

#endif /* MEMIO_H__ */
```

If we then use `mem_read32` in our example, the resulting code does what we need:

```
00000000 <test>:
   0:	e92d4010 	push	{r4, lr}
   4:	ebfffffe 	bl	0 <get_gpio>
   8:	e2800094 	add	r0, r0, #148	; 0x94
   c:	e5903000 	ldr	r3, [r0]
  10:	e3130101 	tst	r3, #1073741824	; 0x40000000
  14:	1afffffc 	bne	c <test+0xc>
  18:	e8bd8010 	pop	{r4, pc}
```

Since we won't be able to directly access the registers using the structure, we'll just define constants for each register with the offset of the register to the base address of the peripheral. An added benefit is that we don't need to remember to use `__attribute__((packed))` to define the structure, and we don't need to add `reserved` fields where there's a hole in the peripheral's register file.

## Peripheral Base Address

The Raspberry Pi has a peripheral named Auxiliary, which has one "mini" [UART](https://en.wikipedia.org/wiki/Universal_asynchronous_receiver-transmitter) (it lacks some functionalities), and two [SPI](https://en.wikipedia.org/wiki/Serial_Peripheral_Interface) masters. It's fully documented in BCM2835 ARM Peripherals, but it has a hidden register that produces `0x61757830` (the ASCII characters `"aux0"`) when read. By [probing this register](https://www.raspberrypi.org/forums/viewtopic.php?t=168898#p1085504) we can find the base address of all memory-mapped peripherals, since they're all at fixed distance from one another.

Lets add a function to `memio` to initialize and retrieve the peripherals' base address:

```c
#ifndef MEMIO_H__
#define MEMIO_H__

#include <stdint.h>

extern const uint32_t g_baseio;

int memio_init(void);

// mem_read32 and mem_write32 omitted.

#endif /* MEMIO_H__ */
```

```c
#include <stdint.h>

uint32_t g_baseio;

static inline uint32_t read32(const uint32_t address) {
  uint32_t result;

  __asm volatile(
    "ldr %[result], [%[address]]\n"
    : [result] "=r" (result)
    : [address] "r" (address)
  );

  return result;
}

int memio_init(void) {
  // Test for peripherals at 0x20000000.
  if (read32(UINT32_C(0x20215010)) == UINT32_C(0x61757830)) {
    g_baseio = UINT32_C(0x20000000);
    return 0;
  }

  // Test for peripherals at 0x3f000000.
  if (read32(UINT32_C(0x3f215010)) == UINT32_C(0x61757830)) {
    g_baseio = UINT32_C(0x3f000000);
    return 0;
  }

  return -1;
}
```

Lets not forget to add `memio.c` to the Makefile:

```
OBJS = kmain.o memio.o
```

Notice that in the header, `g_baseio` is declared as `const`, but not in the C file. As long as we don't include the header in the `.c`, the compiler doesn't know any better than to allocate a symbol named `g_baseio` with four bytes in the `.bss` section in `memio.o`. Other compilation units that include `memio.h` and use the symbol will only have an entry for it as *undefined*, and relocation records to patch addresses where needed.

The linker in its turn will happily resolve every compilation unit looking for this symbol, because it doesn't have any information about it being `const` or not, for all it knows it's only a symbol.

```
$ arm-none-eabi-objdump -x memio.o

SYMBOL TABLE:
00000004       O *COM*	00000004 g_baseio

$ arm-none-eabi-objdump -x kmain.o

SYMBOL TABLE:
00000000         *UND*	00000000 g_baseio


RELOCATION RECORDS FOR [.text]:
OFFSET   TYPE              VALUE 
0000003c R_ARM_ABS32       g_baseio
```

Now we can use `g_baseio` whenever we need the peripherals base address, but we can't inadvertently change it.

Now that the environment for C code fully setup, and we know the base address of the peripherals, we can use the GPIO to blink the LED. Or can we?

## Source Code

The source code listed here can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones02).
