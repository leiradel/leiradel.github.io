---
layout: post
title: Blinking the LED Part 3 - Using the GPIO
---

## Using the GPIO

Having the base address of the peripherals and the GPIO memory layout, we can program the LED pin to turn it on and off. We'll need to use a couple of GPIO registers to make that happen:

* `GPFSEL0` to `GPFSEL5` to program the LED pin for output
* `GPSET0` and `GPSET1` to set the pin high or low

Lets write some code to help us deal with the GPIO.

```c
#ifndef GPIO_H__
#define GPIO_H__

typedef enum {
  GPIO_INPUT      = 0,
  GPIO_OUTPUT     = 1,
  GPIO_FUNCTION_0 = 4,
  GPIO_FUNCTION_1 = 5,
  GPIO_FUNCTION_2 = 6,
  GPIO_FUNCTION_3 = 7,
  GPIO_FUNCTION_4 = 3,
  GPIO_FUNCTION_5 = 2
}
gpio_function_t;

void gpio_select(const unsigned pin, const gpio_function_t mode);
void gpio_set(const unsigned pin, const int high);

#endif /* GPIO_H__ */
```

> The `GPIO_FUNCTION_X` values are used to program certain GPIO pins for use with other peripherals, i.e. the mini UART.

```c
#include "gpio.h"
#include "memio.h"

#include <stdint.h>

#define GPFSEL0   0x00
#define GPFSEL1   0x04
#define GPFSEL2   0x08
#define GPFSEL3   0x0c
#define GPFSEL4   0x10
#define GPFSEL5   0x14
#define GPSET0    0x1c
#define GPSET1    0x20
#define GPCLR0    0x28
#define GPCLR1    0x2c
#define GPLEV0    0x34
#define GPLEV1    0x38
#define GPEDS0    0x40
#define GPEDS1    0x44
#define GPREN0    0x4c
#define GPREN1    0x50
#define GPFEN0    0x58
#define GPFEN1    0x5c
#define GPHEN0    0x64
#define GPHEN1    0x68
#define GPLEN0    0x70
#define GPLEN1    0x74
#define GPAREN0   0x7c
#define GPAREN1   0x80
#define GPAFEN0   0x88
#define GPAFEN1   0x8c
#define GPPUD     0x94
#define GPPUDCLK0 0x98
#define GPPUDCLK1 0x9c

// The GPIO base displacement from the base IO address.
#define BASE_ADDR (g_baseio + UINT32_C(0x00200000))

void gpio_select(const unsigned pin, const gpio_function_t mode) {
  // The register index starting at GPFSEL0.
  const unsigned index = pin / 10;
  // Amount to shift to get to the required pin bits.
  const unsigned shift = (pin % 10) * 3;

  // Calculate the GPFSEL register that contains the configuration for
  // the pin.
  const uint32_t gpfsel = BASE_ADDR + GPFSEL0 + index * 4;

  // Read the register.
  const uint32_t value = mem_read32(gpfsel);

  // Set the desired function for the pin.
  const uint32_t masked = value & ~(UINT32_C(7) << shift);
  const uint32_t fsel = masked | mode << shift;

  // Write the value back to the register.
  mem_write32(gpfsel, fsel);
}

void gpio_set(const unsigned pin, const int high) {
  // The register index starting at GPSET0 or GPCLR0.
  const unsigned index = pin >> 5;
  // The bit in the registers to set or clear the pin.
  const uint32_t bit = UINT32_C(1) << (pin & 31);

  if (high) {
    // Write the bit to GPSEL to set the pin high.
    mem_write32(BASE_ADDR + GPSET0 + index * 4, bit);
  }
  else {
    // Write the bit to GPCLR to set the pin low.
    mem_write32(BASE_ADDR + GPCLR0 + index * 4, bit);
  }
}
```

`gpio_select` will set the given pin to the given function or mode, using the registers `GPFSEL0` to `GPFSEL5`. Each pin can be in one of eight different modes as the `gpio_function_t` enumeration shows. `GPIO_INPUT` and `GPIO_OUTPUT` are self-explanatory, the other modes are documented in [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), section 6.2 - Alternative Function Assignments.

Since we need three bits for the mode of a pin, each `GPFSEL` register can hold the modes of 10 registers. For 54 pins, we need 6 registers. In `gpio_select`, `index` is the `GPFSEL` register that has the mode bits for the pin, and `shift` is the shift amount to get to these bits. With them we can read the register, mask out the mode for the pin, put the new mode there, and write the value back to the register. All the code between `mem_read32` and `mem_write32` should be guarded by a mutex, but we'll do without it for now.

> I wonder if we will be able to use `ldrex` ([Load Register Exclusive](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890318276.html)) and `strex` ([Store Register Exclusive](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890604489.html)) instead of a mutex when the time comes.

`gpio_set` will set an output pin low or high. The `GPSET` registers are used to set a pin high, all we have to do is set the bit that corresponds to the pin. The same thing happens with the `GPCLR` registers, but they're used to set a pin low. Since there's only one bit for each pin, there are only two `GPSET` and two `GPCLR` registers.

> Notice how having different registers to set a pin high and low make it easier to write this code. We won't even need a mutex here. If we had the pin states in a register, we'd had to read the state, set or clear the bit for the pin, and write it back, just like in `gpio_select`.

Now if we add `gpio.c` to the Makefile and build, we'll get an odd error:

```
arm-none-eabi-ld -o kernel.elf -Map kernel.map -T link.T kmain.o memio.o gpio.o
gpio.o: In function `gpio_select':
gpio.c:(.text+0x10): undefined reference to `__aeabi_uidivmod'
gpio.c:(.text+0x20): undefined reference to `__aeabi_uidiv'
Makefile:34: recipe for target 'kernel.img' failed
make: *** [kernel.img] Error 1
```

We surely are *not* calling `__aeabi_uidivmod` nor `__aeabi_uidiv` anywhere, so why is the linker complaining? Lets see the disassembly of `gpio.o` (with uninteresting parts omitted):

```
$ arm-none-eabi-objdump -D gpio.o

00000000 <gpio_select>:
   0:	e92d4070 	push	{r4, r5, r6, lr}
   4:	e1a05001 	mov	r5, r1
   8:	e3a0100a 	mov	r1, #10
   c:	e1a06000 	mov	r6, r0
  10:	ebfffffe 	bl	0 <__aeabi_uidivmod>
  14:	e1a00006 	mov	r0, r6
  18:	e0814081 	add	r4, r1, r1, lsl #1
  1c:	e3a0100a 	mov	r1, #10
  20:	ebfffffe 	bl	0 <__aeabi_uidiv>
  24:	e59f3020 	ldr	r3, [pc, #32]	; 4c <gpio_select+0x4c>
  28:	e5933000 	ldr	r3, [r3]
  2c:	e2833602 	add	r3, r3, #2097152	; 0x200000
  30:	e0830100 	add	r0, r3, r0, lsl #2
  34:	e5901000 	ldr	r1, [r0]
  38:	e3a03007 	mov	r3, #7
  3c:	e1c11413 	bic	r1, r1, r3, lsl r4
  40:	e1811415 	orr	r1, r1, r5, lsl r4
  44:	e5801000 	str	r1, [r0]
  48:	e8bd8070 	pop	{r4, r5, r6, pc}
  4c:	00000000 	andeq	r0, r0, r0
```

It turns out the `udiv` ([Unsigned Divide](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425915385325.html)) instruction is not available in ARMv6. It's the compiler's responsibility to generate code for the architectures it supports, so it must provide a way to make integer division. Compilers implement this as builtin functions as documented in [Run-time ABI for the ARM Architecture](https://static.docs.arm.com/ihi0043/d/IHI0043D_rtabi.pdf), section 4.3.1 - Integer (32/32 -> 32) division functions.

> It's interesting to note that, despite `__aeabi_uidivmod` returning both the quotient and the reminder, the compiler calls `__aeabi_uidiv` to get compute the exact same division again and get the quotient.

Since we decided to leave out standard libraries and builtin functions for this bare-bones project, the linker cannot find the required functions to make the integer division for ARMv6, which we need in `gpio_select`. We could either add these functions from gcc or llvm to our project, or write integer division functions ourselves. We will do neither.

## Divide by 10

In `gpio_select`, `pin` can only have values between 0 and 53, inclusive. These are the available GPIO pins in Raspberry Pi boards. We can accurately compute the integer division of these values by 10 using [fixed-point arithmetic](https://en.wikipedia.org/wiki/Fixed-point_arithmetic), and turning a division into a multiplication by the inverse.

We can decide ourselves how many digits we'll use for the fractional part of the number. Since we're on a 32-bit architecture, let's avoid needing a 64-bit result of the multiplication because that needs multiple instructions to achieve in an ARMv6. This means we're limited to 16 bits for each operand. Since 1/10 doesn't have an integer part, we'll use all those bits as the fractional part, i.e. our fixed-point number will be a 0.16 one.

To calculate what 1/10 in 0.16 fixed-point we just compute 65536 * 1/10, which results in 6553.6. Getting rid of the decimal fraction results in 6554, or 0x199a. So to divide by 10, we instead multiply by 0x199a, but this will of course *not* give us the correct result.

We're representing 1/10 as a 0.16 number. The pin number is a regular integer, so its a 32.0 number. If we remember how we do multiplications on paper, we'll see that those two numbers will result in a 32.16 number when multiplied, so we must get rid of the fractional part in the lower 16 bits to get the integer quotient.

> Since the result of the multiplication will be truncated to 32 bits, the result is actually 16.16, and the quotient of our pin numbers will fit comfortably there.

Lets change `gpio_select`:

```c
void gpio_select(const unsigned pin, const gpio_function_t mode) {
  // The register index starting at GPFSEL0.
  const unsigned index = (pin * 0x199aU) >> 16;
  // Amount to shift to get to the required pin bits.
  const unsigned shift = (pin - index * 10) * 3;

  ...
}
```

If we build now we'll see that the link errors are gone, which is great, but lets see the generated code:

```
00000000 <gpio_select>:
   0:	e59f303c 	ldr	r3, [pc, #60]	; 44 <gpio_select+0x44>
   4:	e0030093 	mul	r3, r3, r0
   8:	e1a02823 	lsr	r2, r3, #16
   c:	e3a0300a 	mov	r3, #10
  10:	e0030293 	mul	r3, r3, r2
  14:	e0400003 	sub	r0, r0, r3
  18:	e59f3028 	ldr	r3, [pc, #40]	; 48 <gpio_select+0x48>
  1c:	e0800080 	add	r0, r0, r0, lsl #1
  20:	e5933000 	ldr	r3, [r3]
  24:	e2833602 	add	r3, r3, #2097152	; 0x200000
  28:	e0833102 	add	r3, r3, r2, lsl #2
  2c:	e5932000 	ldr	r2, [r3]
  30:	e3a0c007 	mov	ip, #7
  34:	e1c2201c 	bic	r2, r2, ip, lsl r0
  38:	e1820011 	orr	r0, r2, r1, lsl r0
  3c:	e5830000 	str	r0, [r3]
  40:	e12fff1e 	bx	lr
  44:	0000199a 	muleq	r0, sl, r9
  48:	00000000 	andeq	r0, r0, r0
```

The builtin function calls are gone, but the compiler is putting the `0x199a` constant in memory right after the end of the function, and loading it with a `ldr` ([Load with immediate offset](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890250491.html)). Unfortunately, the `mov` ([Move](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890440026.html)) instruction on an ARMv6 CPU doesn't work with 16-bit immediate values, only with [8-bit immediate values with a shift](http://www.davespace.co.uk/arm/introduction-to-arm/operand2.html). We'll have to try something different.

Lets write a program to find all the fixed-point numbers that represent 1/10 and give the correct results for our input domain, 0 to 53:

```c
#include <stdio.h>

static int test(unsigned magic, unsigned shift) {
  unsigned i, a, b;

  for (i = 0; i < 54; i++) {
    a = i / 10;
    b = (i * magic) >> shift;

    if (a != b) {
      return 0;
    }
  }

  return 1;
}

int main() {
  unsigned magic, shift;

  for (shift = 0; shift < 29; shift++) {
    magic = ((1 << shift) + 5) / 10;
    printf("(pin * 0x%08xU) >> %2u => %d\n", magic, shift, test(magic, shift));
  }

  return 0;
}
```

This program tries fixed-point numbers from 32.0 to 3.29, since we need at least 3 bits to hold the biggest quotient for our input domain, 5. Lets run it:

```
$ ./div10tst1 |grep "=> 1"
(pin * 0x0000000dU) >>  7 => 1
(pin * 0x0000001aU) >>  8 => 1
(pin * 0x000000cdU) >> 11 => 1
(pin * 0x0000019aU) >> 12 => 1
(pin * 0x00000ccdU) >> 15 => 1
(pin * 0x0000199aU) >> 16 => 1
(pin * 0x0000cccdU) >> 19 => 1
(pin * 0x0001999aU) >> 20 => 1
(pin * 0x000ccccdU) >> 23 => 1
(pin * 0x0019999aU) >> 24 => 1
(pin * 0x00cccccdU) >> 27 => 1
(pin * 0x0199999aU) >> 28 => 1
```

> Not all combinations pass the test. The ones that fail will pass if we increment the magic number. The answer is [here](http://ridiculousfish.com/blog/posts/labor-of-division-episode-i.html), somewhere...

> It's interesting to see that, if we change the program to do 64-bit multiplications and go up to 32.32, it will find the same answer as [here](https://stackoverflow.com/questions/5558492/divide-by-10-using-bit-shifts).

The greatest magic number that fits in eight bits is `0xcd`, so lets use it and see the code that it generates:

```c
unsigned div10_cd(unsigned i) {
  return (i * 0xcdU) >> 11;
}
```

When we compile `div10_cd` using the same flags as for `gpio.c` and disassemble this, we find that `0xcd` is indeed the answer:

```
$ arm-none-eabi-objdump -D div10tst2.o 

00000028 <div10_cd>:
  28:	e3a030cd 	mov	r3, #205	; 0xcd
  2c:	e0000093 	mul	r0, r3, r0
  30:	e1a005a0 	lsr	r0, r0, #11
  34:	e12fff1e 	bx	lr
```

Lets put that in `gpio.c` and take a last look at the generated code:

```c
void gpio_select(const unsigned pin, const gpio_function_t mode) {
  // The register index starting at GPFSEL0.
  const unsigned index = (pin * 0xcdU) >> 11;
  // Amount to shift to get to the required pin bits.
  const unsigned shift = (pin - index * 10) * 3;

  ...
}
```

```
$ arm-none-eabi-objdump -D gpio.o

00000000 <gpio_select>:
   0:	e3a030cd 	mov	r3, #205	; 0xcd
   4:	e0030093 	mul	r3, r3, r0
   8:	e1a025a3 	lsr	r2, r3, #11
   c:	e3a0300a 	mov	r3, #10
  10:	e0030293 	mul	r3, r3, r2
  14:	e0400003 	sub	r0, r0, r3
  18:	e59f3024 	ldr	r3, [pc, #36]	; 44 <gpio_select+0x44>
  1c:	e0800080 	add	r0, r0, r0, lsl #1
  20:	e5933000 	ldr	r3, [r3]
  24:	e2833602 	add	r3, r3, #2097152	; 0x200000
  28:	e0833102 	add	r3, r3, r2, lsl #2
  2c:	e5932000 	ldr	r2, [r3]
  30:	e3a0c007 	mov	ip, #7
  34:	e1c2201c 	bic	r2, r2, ip, lsl r0
  38:	e1820011 	orr	r0, r2, r1, lsl r0
  3c:	e5830000 	str	r0, [r3]
  40:	e12fff1e 	bx	lr
  44:	00000000 	andeq	r0, r0, r0
```

Great, we don't need the builtin function for integer divisions anymore, and our division doesn't need to load a constant from memory.

Now that we have the necessary functions to deal with the GPIO, we can finally make the LED blink. Right?

## Source Code

The source code listed here can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones03).
