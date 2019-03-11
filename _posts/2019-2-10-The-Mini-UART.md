---
layout: post
title: The Mini UART
---

Blinking the LED is a great baremetal Hello World program, but it severely limits our interaction with the program that is running in the board. We want to be able to see text coming out of it, and send keystrokes to it. Using a serial connection between the board and a PC is the easiest way to achieve that.

While connecting the board to a TV set and using the framebuffer for text is not difficult, reading keys from an USB keyboard requires writing an USB driver, which is not exactly trivial. Open source USB drivers for the Raspberry Pi boards exist, but there's something else that makes the serial connection attractive.

Testing baremetal programs becomes tiresome very fast, having to swap SDRAM cards and copy things all the time -- not counting the strain imposed to the board, the PC card reader, and the card itself. If we can transfer programs via a serial cable, we can write a kernel that sets up the environment and loads binaries from the UART and runs them.

> Note: I bought this [USB to TTL Serial Cable](https://www.ptrobotics.com/microcontroladores-nxp/2330-usb-to-ttl-serial-cable-debug-console-cable-for-raspberry-pi.html) ([similar product at Adafruit](https://www.adafruit.com/product/954)) to connect the Mini UART to my PC. Make sure that it uses 3.3V on the receive and transmit pins, as 5V will likely burn your Raspberry Pi board. Also, if it has a 5V power pin, you can connect it to the board to power it up without the need of a regular USB cable.

## The Mini UART

The Raspberry Pi boards feature an almost [16650](https://en.wikipedia.org/wiki/16550_UART)-compatible UART called the Mini UART. This UART is documented at [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), section 2.2 - Mini UART, and is part of the auxiliary peripherals, together with SPI1 and SPI2.

[It uses GPIO pins 14 and 15](https://pinout.xyz/pinout/uart#) for transmit (`TXD`) and receive (`RXD`), respectively. However, we have to tell the GPIO that we want to use those pins for the UART, otherwise they will be just general I/O pins. BCM2835 ARM Peripherals, section 6.2 - Alternative Function Assignments lists all the alternative functions of all the GPIO pins. Looking for `TXD0` and `RXD0` (`TXD1` and `RXD1` are the pins for the full UART, a different peripheral) we can see that they are pins 14 and 15 in the alternative function ALT0.

> `TXD0` and `RXD0` are also exposed in pins 32 and 33 in alternative function ALT3, and pins 36 and 37 in alternative function ALT2. We have multiple options because we can decide to use other pins if we're already using 14 and 15 for something else, i.e. the full UART.

## Configuring Pull-Up/Down

Besides selecting the appropriate alternative function, we also have to turn off the pull up/down resistors for these GPIO pins. This is something we have neglected until now.

The procedure to change the pull-up/down is described at BCM2835 ARM Peripherals, page 101, in the description of the `GPPUDCLK` registers. It says we must wait 150 cycles to allow for the required control signal setup time. In the absence of more details, we'll just run an empty loop for 150 iterations, which will give us more than we need.

```c
typedef enum {
  GPIO_PULL_OFF  = 0,
  GPIO_PULL_DOWN = 1,
  GPIO_PULL_UP   = 2
}
gpio_pull_t;

void gpio_setpull(const unsigned pin, const gpio_pull_t pull);
```

```c
static void wait(unsigned count) {
  // Spend CPU cycles.
  while (count-- != 0) {
    // Empty.
  }
}

void gpio_setpull(const unsigned pin, const gpio_pull_t pull) {
  const unsigned index = pin >> 5;
  const uint32_t bit = UINT32_C(1) << (pin & 31);

  // Set GPPUD to the desired pull up/down.
  mem_write32(BASE_ADDR + GPPUD, pull);
  // Spend cycles.
  wait(150);

  // Evaluate which GPPUDCLK register is needed.
  const uint32_t gppudclk = BASE_ADDR + (index == 0 ? GPPUDCLK0 : GPPUDCLK1);

  // Set the pin bit in GPPUDCLK.
  mem_write32(gppudclk, bit);
  // Spend cycles.
  wait(150);
  // Clear the bit.
  mem_write32(gppudclk, 0);
}
```

> It's a good idea to also change `led.c` and use `gpio_setpull` to set the activity LED pin to `GPIO_PULL_OFF`.

## Using the UART

Now we can write code to initialize the UART and interact with a terminal running at the other end of the serial cable.

```c
#ifndef AUX_H__
#define AUX_H__

void uart_init(const unsigned baudrate);

int uart_canwrite(void);
int uart_write(const char k, unsigned timeout_us);

int uart_canread(void);
int uart_read(unsigned timeout_us);

#endif /* AUX_H__ */
```

```c
#include "gpio.h"
#include "memio.h"
#include "timer.h"

#include <stdint.h>

#define AUX_IRQ         0x00
#define AUX_ENABLES     0x04
#define AUX_MU_IO_REG   0x40
#define AUX_MU_IER_REG  0x44
#define AUX_MU_IIR_REG  0x48
#define AUX_MU_LCR_REG  0x4c
#define AUX_MU_MCR_REG  0x50
#define AUX_MU_LSR_REG  0x54
#define AUX_MU_MSR_REG  0x58
#define AUX_MU_SCRATCH  0x5c
#define AUX_MU_CNTL_REG 0x60
#define AUX_MU_STAT_REG 0x64
#define AUX_MU_BAUD_REG 0x68

#define BASE_ADDR (g_baseio + UINT32_C(0x00215000))

void uart_init(const unsigned baudrate) {
  // Set pins 14 and 15 to use with the Mini UART.
  gpio_select(14, GPIO_FUNCTION_5);
  gpio_select(15, GPIO_FUNCTION_5);

  // Turn pull up/down off for those pins.
  gpio_setpull(14, GPIO_PULL_OFF);
  gpio_setpull(15, GPIO_PULL_OFF);

  // Enable only the Mini UART.
  mem_write32(BASE_ADDR + AUX_ENABLES, 1);
  // Turn off interrupts, we'll use polling for now.
  mem_write32(BASE_ADDR + AUX_MU_IER_REG, 0);
  // Disable receiving and transmitting while we configure the Mini UART.
  mem_write32(BASE_ADDR + AUX_MU_CNTL_REG, 0);
  // Set data size to 8 bits.
  mem_write32(BASE_ADDR + AUX_MU_LCR_REG, 1);
  // Put RTS high.
  mem_write32(BASE_ADDR + AUX_MU_MCR_REG, 0);
  // Clear both receive and transmit FIFOs.
  mem_write32(BASE_ADDR + AUX_MU_IIR_REG, 6);
  // Set the desired baudrate.
  const uint32_t divisor = 250000000 / (8 * baudrate) - 1;
  mem_write32(BASE_ADDR + AUX_MU_BAUD_REG, divisor);
  // Enable receiving and transmitting.
  mem_write32(BASE_ADDR + AUX_MU_CNTL_REG, 3);
}

int uart_canwrite(void) {
  const uint32_t value = mem_read32(BASE_ADDR + AUX_MU_LSR_REG);
  return (value & 0x20) != 0;
}

int uart_write(const int k, unsigned timeout_us) {
  if (timeout_us == 0) {
    // timeout_us == 0 means retry forever.
    while (!uart_canwrite()) {
      // nothing
    }
  }
  else {
    uint64_t timeout = timer() + timeout_us;

    while (!uart_canwrite()) {
      if (timer() >= timeout) {
        return -1;
      }
    }
  }

  // There's space in the transmit FIFO, write the character
  mem_write32(BASE_ADDR + AUX_MU_IO_REG, k & 0xff);
  return 0;
}

int uart_canread(void) {
  const uint32_t value = mem_read32(BASE_ADDR + AUX_MU_LSR_REG);
  return value & 1;
}

int uart_read(unsigned timeout_us) {
  if (timeout_us == 0) {
    // timeout_us == 0 means retry forever.
    while (!uart_canread()) {
      // nothing
    }
  }
  else {
    uint64_t timeout = timer() + timeout_us;

    while (!uart_canread()) {
      if (timer() >= timeout) {
        return -1;
      }
    }
  }

  // There's data available in the receive FIFO, return it.
  return mem_read32(BASE_ADDR + AUX_MU_IO_REG) & 0xff;
}
```

Note that we use `250000000 / (8 * baudrate) - 1` to calculate the divisor that should go into the `AUX_MU_BAUD_REG` register to set the UART speed. This is a pretty direct implementation of what is documented in BCM2835 ARM Peripherals, but we have a division where the divisor is not a constant.

We could deal with that by forcing the programmer to pass the divisor to the `uart_init` function, and maybe even provide a macro to turn the division in runtime into one in compile time when the baudrate is a constant. However, I believe its time to have the builtin functions available.

## Compiler-RT

The [Compiler-RT](http://compiler-rt.llvm.org/) project implements, among other things, the built-in functions required by a number of architectures, ARM included as documented in [Run-time ABI for the ARM Architecture](https://static.docs.arm.com/ihi0043/d/IHI0043D_rtabi.pdf). A mirror of the source code is available at [GitHub](https://github.com/llvm-mirror/compiler-rt), and it's released under the Apache License 2.0.

After fighting a bit with `cmake`, I decided to write a simple Makefile that builds the builtin functions into a library we can link to. The Makefile is included in the source code as `Compiler-RT.makefile`.

We'll build one library for each of the three architectures we care about, and copy them into our project directory under `lib/v6`, `lib/v7`, and `lib/v8-32`.

```
$ git clone --recursive https://github.com/llvm-mirror/compiler-rt.git
$ cd compiler-rt/lib/builtins

$ make -f Compiler-RT.makefile clean
$ make -f Compiler-RT.makefile -j
$ mkdir <path-to-project>/lib/v6
$ cp librt_v6.a <path-to-project>/lib/v6/librt.a

$ make -f Compiler-RT.makefile clean
$ make v7=1 -f Compiler-RT.makefile -j
$ mkdir <path-to-project>/lib/v7
$ cp librt_v7.a <path-to-project>/lib/v7/librt.a

$ make -f Compiler-RT.makefile clean
$ make v8=1 -f Compiler-RT.makefile -j
$ mkdir <path-to-project>/lib/v8-32
$ cp librt_v8-32.a <path-to-project>/lib/v8-32/librt.a
```

We should have this hierarchy in the project directory now.

* `lib/`
    * `v6/`
        * `librt.a`
    * `v7/`
        * `librt.a`
    * `v8-32/`
        * `librt.a`

Let's also change the baremetal Makefile to select the appropriate lib depending on the target:

```
ifeq ($(strip $(v7)),1)
  CFLAGS  += -march=armv7-a -mtune=cortex-a7 -marm -mfpu=neon-vfpv4 -mfloat-abi=hard
  LDFLAGS += -Llib/v7
  TARGET   = kernel7
else ifeq ($(strip $(v8)),1)
  CFLAGS  += -march=armv8-a -mtune=cortex-a53 -marm -mfpu=neon-fp-armv8 -mfloat-abi=hard
  LDFLAGS += -Llib/v8-32
  TARGET   = kernel8-32
else
  CFLAGS  += -march=armv6k -mtune=arm1176jzf-s -marm -mfpu=vfp -mfloat-abi=hard
  LDFLAGS += -Llib/v6
  TARGET   = kernel
endif
```

And lets also add the library to the linker command line:

```
$(LD) -o $(TARGET).elf -Map $(TARGET).map $(LDFLAGS) $(OBJS) -lrt
```

If we now build the kernel with the division in `uart_init`, the linker will find `__aeabi_uidiv` in `librt.a` and everything works as expected.

## libc

Now that we have the builtin functions, let's also build a `libc` and have all the facilities that it provides. It'll be awesome to write to the UART using `printf`.

[Newlib](https://sourceware.org/newlib/) is a well-known libc. It's licensing is a bit messy (check `COPYING.NEWLIB`) but should be ok for us. It's also super-easy to build, and I've included a `Newlib-build.sh` script to build for all three architectures in the source code.

> [musl](http://www.musl-libc.org/) has cleaner license terms, but I couldn't build it without it being tied to the Linux syscalls, which goes rounds to make a kernel call via `svc #0` ([SuperVisor Call](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425914052313.html)). Newlib in its turn provides a build where the syscalls are just regular calls to functions we can implement directly in C.

```
$ git clone --recursive git://sourceware.org/git/newlib-cygwin.git
$ cd newlib-cygwin
$ sh <path-to-Newlib-build.sh>
```

At the end, the `newlib-cygwin` directory will contain one subdirectory for each architecture: `v6`, `v7`, and `v8-32`, each with the subdirectories `arm-none-eabi/include` and `arm-none-eabi/lib`.

Let's copy the libc and libm libraries to their respective directories under the `lib` directory, just like we did with `librt.a`, so we have this:

* `lib/`
    * `v6/`
        * `libc.a`
        * `libm.a`
        * `librt.a`
    * `v7/`
        * `libc.a`
        * `libm.a`
        * `librt.a`
    * `v8-32/`
        * `libc.a`
        * `libm.a`
        * `librt.a`

Let's also copy the files in `include` to an `include` directory in our baremetal project, like this:

* `include/`
    * `v6/`
        * Header files
    * `v7/`
        * Header files
    * `v8-32/`
        * Header files

Make sure to compile the header files from the corresponding architecture, just to make sure nothing will break because headers are being used with the wrong libraries.

Now that we have the Newlib headers, we can remove the ones we copied from the `arm-none-eabi` toolchain, `stddef.h`, `stdint-gcc.h`, and `stdint.h`. However, and rather surprisingly to me, the Newlib headers still don't include all that we need. We'll copy them again from the toolchain, but this time into the `include/common` directory:

```
$ mkdir include/common
$ cp /usr/lib/gcc/arm-none-eabi/6.3.1/include/stdarg.h include/common
$ cp /usr/lib/gcc/arm-none-eabi/6.3.1/include/stddef.h include/common
```

Finally, we update the Makefile to add the correct include directory, and to link to libc and libm. That's how it should look now:

```
PREFIX = arm-none-eabi-
CC = $(PREFIX)gcc
CXX = $(PREFIX)g++
LD = $(PREFIX)ld
OBJDUMP = $(PREFIX)objdump
OBJCOPY = $(PREFIX)objcopy
CPPFILT = $(PREFIX)c++filt

CFLAGS  = -fsigned-char -ffreestanding -nostdinc -std=c99
DEFINES = -DBOARD_STRINGS
LDFLAGS = -T link.T

ifeq ($(strip $(v7)),1)
  CFLAGS  += -march=armv7-a -mtune=cortex-a7 -marm -mfpu=neon-vfpv4 -mfloat-abi=hard
  CFLAGS  += -Iinclude/v7
  LDFLAGS += -Llib/v7
  TARGET   = kernel7
else ifeq ($(strip $(v8)),1)
  CFLAGS  += -march=armv8-a -mtune=cortex-a53 -marm -mfpu=neon-fp-armv8 -mfloat-abi=hard
  CFLAGS  += -Iinclude/v8-32
  LDFLAGS += -Llib/v8-32
  TARGET   = kernel8-32
else
  CFLAGS  += -march=armv6k -mtune=arm1176jzf-s -marm -mfpu=vfp -mfloat-abi=hard
  CFLAGS  += -Iinclude/v6
  LDFLAGS += -Llib/v6
  TARGET   = kernel
endif

ifneq ($(DEBUG),)
  CFLAGS += -O0 -g
else
  CFLAGS += -Os -DNDEBUG
endif

%.o: %.c
	$(CC) $(CFLAGS) $(DEFINES) $(INCLUDES) -c $< -o $@

%.o: %.S
	$(CC) $(CFLAGS) $(DEFINES) $(INCLUDES) -c $< -o $@

OBJS = kmain.o memio.o gpio.o mbox.o prop.o board.o led.o timer.o aux.o glod.o irq.o main.o

all: $(TARGET).img

$(TARGET).img: $(OBJS)
	$(LD) -o $(TARGET).elf -Map $(TARGET).map $(LDFLAGS) $(OBJS) -lc -lm -lrt
	$(OBJDUMP) -d $(TARGET).elf | $(CPPFILT) > $(TARGET).lst
	$(OBJCOPY) $(TARGET).elf -O binary $(TARGET).img
	wc -c $(TARGET).img

clean:
	rm -f $(TARGET).elf $(TARGET).map $(TARGET).lst $(TARGET).img $(OBJS)

.PHONY: clean
```

Let's now implement the syscalls Newlib needs to work.

```c
#include <stdint.h>
#include <unistd.h>
#include <sys/stat.h>

#include "aux.h"

// Extend the heap space at the end of the .bss section.
void* _sbrk(intptr_t increment) {
  extern char heap_start;
  static const char* start = &heap_start;
  const char* prev = start;

  // Keep memory returned by _sbrk 16-byte aligned.
  increment = (increment + 15) & ~15;
  start += increment;

  return (void*)prev;
}

// Write stdout and stderr to the UART.
ssize_t _write(int fd, const void* buf, size_t count) {
  if (fd == 1 || fd == 2) {
    const char* str = buf;

    for (size_t i = 0; i < count; i++) {
      if (*str == '\n') {
        uart_write('\r', 0);
      }

      uart_write(*str++, 0);
    }

    return count;
  }

  return 0;
}

// Read stdin from the UART.
ssize_t _read(int fd, void* buf, size_t count) {
  if (fd == 0) {
    char* str = buf;

    for (size_t i = 0; i < count; i++) {
      *str++ = uart_read(0);
    }

    return count;
  }

  return 0;
}

int _close(int fd) {
  (void)fd;
  return 0;
}

int _fstat(int fd, struct stat* buf) {
  (void)fd;
  (void)buf;
  return 0;
}

int _isatty(int fd) {
  (void)fd;
  return 1;
}

off_t _lseek(int fd, off_t offset, int whence) {
  (void)fd;
  (void)offset;
  (void)whence;
  return 0;
}
```

We'll need to implement other syscalls as we use more libc features, but those will do for now. Let's add `syscalls.o` to out Makefile:

```
OBJS = kmain.o memio.o gpio.o mbox.o prop.o board.o led.o \
       timer.o aux.o glod.o irq.o syscalls.o \
       main.o
```

There's only one thing missing. `_sbrk` works by returning contiguous memory each time it's called. `malloc` and friends use it to get more memory to use for the heap. It's implement using RAM starting right after the end of the `.bss` section, but it uses a symbol that we didn't define yet: `heap_start`.

Let's change our `link.T` file to add this symbol right after `.bss`, aligned to a 16-byte boundary:

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
    bss_start = .;
    *(.bss .bss.*)
    *(COMMON)
    . = ALIGN(4);
    bss_end = .;
  }

  . = ALIGN(16);
  heap_start = .;

  /DISCARD/ : {
    *(*)
  }
}
```

We should also check for the end of usable memory, which is where the VideoCore memory starts, but this will do for now.

Let's write a `main` function that writes some board details to the UART.

## Getting Board Details

Until now, the only property we use from the GPU is the board revision, which we use to identify the board and use the correct GPIO pin to blink the activity LED. There are however a lot more property tags that we can use to get detailed information from the GPU. We'll implement some of them to send detailed board information via the UART.

```c
#ifndef PROP_H__
#define PROP_H__

#include <stdint.h>

#define PROP_CLOCK_EMMC   1
#define PROP_CLOCK_UART   2
#define PROP_CLOCK_ARM    3
#define PROP_CLOCK_CORE   4
#define PROP_CLOCK_V3D    5
#define PROP_CLOCK_H264   6
#define PROP_CLOCK_ISP    7
#define PROP_CLOCK_SDRAM  8
#define PROP_CLOCK_PIXEL  9
#define PROP_CLOCK_PWM   10

typedef struct {
  uint32_t base;
  uint32_t size;
}
memrange_t;

uint32_t   prop_revision(void);
uint32_t   prop_fwrev(void);
uint32_t   prop_model(void);
uint64_t   prop_macaddr(void);
uint64_t   prop_serial(void);
memrange_t prop_armmemory(void);
memrange_t prop_vcmemory(void);
int        prop_cmdline(char cmdline[static 256]);
uint32_t   prop_getclockrate(const uint32_t clock_id);
uint32_t   prop_getminclockrate(const uint32_t clock_id);
uint32_t   prop_getmaxclockrate(const uint32_t clock_id);

#endif /* PROP_H__ */
```

The implementation of those functions are very similar and rather long and tedious, so I'll omit the source code here.

## `main.c`

Since we have libc now, we can just `printf` everything we want to report about our board.

```c
#include "armregs.h"
#include "memio.h"
#include "aux.h"
#include "board.h"
#include "prop.h"

#include <stdio.h>
#include <inttypes.h>

void main(const uint32_t machine_type, const uint32_t atags_addr) {
  uart_init(115200);

  const uint32_t midr = get_midr();
  const uint32_t variant = (midr & MIDR_VARIANT_NO) >> 20;
  const uint32_t revision = midr & MIDR_REVISION;
  const uint32_t part_no = midr & MIDR_PRIMARY_PART_NO;

  if (part_no == MIDR_PART_NO_ARM1176JZF_S) {
    printf("CPU:          ARM1176JZF-S r%up%u\n", variant, revision);
  }
  else if (part_no == MIDR_PART_NO_CORTEX_A7) {
    printf("CPU:          Cortex-A7 r%up%u\n", variant, revision);
  }
  else if (part_no == MIDR_PART_NO_CORTEX_A53) {
    printf("CPU:          Cortex-A53 r%up%u\n", variant, revision);
  }

  const board_t board = board_info(prop_revision());
  printf(
    "Board:        %s %s %u.%u %u %s\n",
    board_model(board.model),
    board_processor(board.processor),
    board.rev_major,
    board.rev_minor,
    board.ram_mb,
    board_manufacturer(board.manufacturer)
  );

  printf("Machine type: 0x%08" PRIx32 "\n", machine_type);
  printf("Tags:         0x%08" PRIx32 "\n", atags_addr);
  printf("Base I/O:     0x%08" PRIx32 "\n", g_baseio);

  const uint32_t fwrev = prop_fwrev();
  const uint32_t model = prop_model();
  const uint32_t rev = prop_revision();
  const uint64_t macaddr = prop_macaddr();
  const uint64_t serial = prop_serial();
  const memrange_t armmem = prop_armmemory();
  const memrange_t vcmem = prop_vcmemory();

  char cmdline[256];
  prop_cmdline(cmdline);

  printf("Firmware:     0x%08" PRIx32 "\n", fwrev);
  printf("Model:        0x%08" PRIx32 "\n", model);
  printf("Revision:     0x%08" PRIx32 "\n", rev);

  printf(
    "Mac addr:     %02x:%02x:%02x:%02x:%02x:%02x\n",
    (macaddr >> 40) & 255,
    (macaddr >> 32) & 255,
    (macaddr >> 24) & 255,
    (macaddr >> 16) & 255,
    (macaddr >> 8) & 255,
    macaddr & 255
  );

  printf("Serial:       0x%016" PRIx64 "\n", serial);

  printf(
    "ARM memory:   base 0x%08" PRIx32 ", size 0x%08" PRIx32 "\n",
    armmem.base,
    armmem.size
  );

  printf(
    "VC memory:    base 0x%08" PRIx32 ", size 0x%08" PRIx32 "\n",
    vcmem.base,
    vcmem.size
  );

  static const uint32_t clock_id[] = {
    PROP_CLOCK_EMMC, PROP_CLOCK_UART, PROP_CLOCK_ARM, PROP_CLOCK_CORE,
    PROP_CLOCK_V3D, PROP_CLOCK_H264, PROP_CLOCK_ISP, PROP_CLOCK_SDRAM,
    PROP_CLOCK_PIXEL, PROP_CLOCK_PWM
  };

  static const char* const clock_name[] = {
    "emmc:", "uart:", "arm:", "core:", "v3d:",
    "h264:", "isp:", "sdram:", "pixel:", "pwm:"
  };

  for (unsigned i = 0; i < sizeof(clock_id) / sizeof(clock_id[0]); i++) {
    const uint32_t rate = prop_getclockrate(clock_id[i]);
    const uint32_t min_rate = prop_getminclockrate(clock_id[i]);
    const uint32_t max_rate = prop_getmaxclockrate(clock_id[i]);

    printf(
      "Clock %-7s %u Hz (%u, %u)\n",
      clock_name[i],
      rate,
      min_rate,
      max_rate
    );
  }
}
```

Running this code in my RPi 1 B board gives this information:

```
$ minicom -b 115200 -D /dev/ttyUSB0

CPU:          ARM1176JZF-S r0p7
Board:        B BCM2835 1.0 256 Egoman
Machine type: 0x00000c42
Tags:         0x00000000
Base I/O:     0x20000000
Firmware:     0x5bdf1ecb
Model:        0x00000000
Revision:     0x00000002
Mac addr:     00:b8:00:27:00:eb
Serial:       0x0000000000000000
ARM memory:   base 0x00000000, size 0x0c000000
VC memory:    base 0x0c000000, size 0x04000000
Clock emmc:   200000000 Hz (200000000, 200000000)
Clock uart:   48000000 Hz (0, 1000000000)
Clock arm:    700000000 Hz (700000000, 700000000)
Clock core:   250000000 Hz (250000000, 250000000)
Clock v3d:    250000000 Hz (250000000, 250000000)
Clock h264:   250000000 Hz (250000000, 250000000)
Clock isp:    250000000 Hz (250000000, 250000000)
Clock sdram:  400000000 Hz (400000000, 400000000)
Clock pixel:  0 Hz (0, 2400000000)
Clock pwm:    0 Hz (0, 500000000)
```

And in my RPi 3 B+ the information is:

```
$ minicom -b 115200 -D /dev/ttyUSB0

CPU:          Cortex-A53 r0p4
Board:        3B+ BCM2837 1.3 1024 Sony UK
Machine type: 0x00000c42
Tags:         0x00000000
Base I/O:     0x3f000000
Firmware:     0x5bdf1ecb
Model:        0x00000000
Revision:     0x00a020d3                                
Mac addr:     00:b8:00:27:00:eb                         
Serial:       0x0000000000000000                        
ARM memory:   base 0x00000000, size 0x3b400000          
VC memory:    base 0x3b400000, size 0x04c00000          
Clock emmc:   200000000 Hz (200000000, 200000000)
Clock uart:   48000000 Hz (0, 1000000000)
Clock arm:    600000000 Hz (600000000, 1400000000)
Clock core:   250000000 Hz (250000000, 400000000)
Clock v3d:    250000000 Hz (250000000, 300000000)
Clock h264:   250000000 Hz (250000000, 300000000)
Clock isp:    250000000 Hz (250000000, 300000000)
Clock sdram:  400000000 Hz (400000000, 450000000)
Clock pixel:  0 Hz (0, 2400000000)
Clock pwm:    0 Hz (0, 500000000)
```

We didn't implement code to receive and run binaries via the serial cable, but we'll do in a future post.

## Source Code

The source code and the compiled libraries can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones07).
