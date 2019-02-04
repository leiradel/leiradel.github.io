---
layout: post
title: Blinking the LED Part 4 - Blinking the LED
---

## Pin 130

Now that we can identify the board, we can use the GPIO functions that we already wrote to set the state of the activity LED. However, there's still one thing to take care of: the pin number 130.

Newer boards have a GPIO expander visible only to the VideoCore, and in the Pi 3 Model B board revision 1.2 the activity LED is connected to a pin in this expander. This expander is not mapped to the ARM address space, so we can't set the activity LED in this board like we can using the regular GPIO peripheral – we must ask the GPU to do it.

The way to ask things to the GPU is via mailbox messages. Since we already wrote a function to send messages to the GPU, we just have to know the tag number value and the buffer format to set the state of a pin of the GPIO expander. The tag number is `0x00038041`, and the tag value is:

```c
typedef union {
  struct {
    uint32_t pin;   // GPIO pin number.
    uint32_t value; // 1 = high, 0 = low.
  }
  request;
  // No response.
}
value_t;
```

## LED Setup

Now that we have everything in place, let's write a small LED module that will initialize things to use the correct GPIO pin to blink the activity LED:

```c
#ifndef LED_H__
#define LED_H__

extern void (*led_set)(const int on);

int led_init(void);

#endif /* LED_H__ */
```

```c
#include "gpio.h"
#include "prop.h"
#include "board.h"
#include "mbox.h"

static void set_16_inv(const int on) {
  gpio_set(16, !on);
}

static void set_47_inv(const int on) {
  gpio_set(47, !on);
}

static void set_130(const int on) {
  typedef struct {
    mbox_msgheader_t header;
    mbox_tagheader_t tag;
    
    union {
      struct {
        uint32_t pin;
        uint32_t value;
      }
      request;
      // No response.
    }
    value;

    mbox_msgfooter_t footer;
  }
  message_t;

  message_t msg __attribute__((aligned(16)));

  msg.header.size = sizeof(msg);
  msg.header.code = 0;
  msg.tag.id = UINT32_C(0x00038041);
  msg.tag.size = sizeof(msg.value);
  msg.tag.code = 0;
  msg.value.request.pin = 130;
  msg.value.request.value = !!on;
  msg.footer.end = 0;

  mbox_send(&msg);
}

static void set_29(const int on) {
  gpio_set(29, on);
}

static void set_none(const int on) {
  (void)on;
}

static void set_47(const int on) {
  gpio_set(47, on);
}

void (*led_set)(const int on) = set_none;

int led_init(void) {
  board_t board;
  const uint32_t revision = prop_get_revision();

  {
    int res;
    
    if ((res = board_info(&board, revision)) != 0) {
      return res;
    }
  }

  if (board.model == BOARD_MODEL_A || board.model == BOARD_MODEL_B) {
    gpio_select(16, GPIO_OUTPUT);
    led_set = set_16_inv;
  }
  else if (board.model == BOARD_MODEL_ZERO ||
           board.model == BOARD_MODEL_ZERO_W) {
    gpio_select(47, GPIO_OUTPUT);
    led_set = set_47_inv;
  }
  else if (board.model == BOARD_MODEL_3B &&
           board.rev_major == 1 &&
           board.rev_minor == 2) {
    // GPIO expander, output only.
    led_set = set_130;
  }
  else if (board.model == BOARD_MODEL_3A_PLUS ||
           board.model == BOARD_MODEL_3B_PLUS) {
    gpio_select(29, GPIO_OUTPUT);
    led_set = set_29;
  }
  else if (board.model == BOARD_MODEL_CM1 || board.model == BOARD_MODEL_CM3) {
    // These boards don't feature an activity LED.
    led_set = set_none;
  }
  else {
    gpio_select(47, GPIO_OUTPUT);
    led_set = set_47;
  }

  return 0;
}
```

## `start`

We have to change our `start` function to initialize the `memio` and `led` modules. Let's take the chance and write a loop that will make the LED blink indefinitely:

```c
static void __attribute__((noinline)) start(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {

  // Zero the .bss section.
  extern uint32_t bss_start, bss_end;

  for (uint32_t* i = &bss_start; i < &bss_end; i++) {
    *i = zero;
  }

  memio_init();
  led_init();

  while (1) {
    led_set(1);

    for (int i = 0; i < 1000000; i++) {
      __asm volatile("");
    }

    led_set(0);
    
    for (int i = 0; i < 1000000; i++) {
      __asm volatile("");
    }
  }
}
```

> The compiler sometimes optimize out the busy loop. To avoid that, we add `__asm volatile("")`. The compiler doesn't know it's empty, and is forced to emit code for the loop. It's also a good idea to make the same thing to the infinite loop at the end of `kmain`.

Now we only need to copy the kernel to a SDRAM card, insert it in a Raspberry Pi, and turn it on, but the card has to be formatted with the correct file system, and some files must be there for the board to boot.

## Preparing the SDRAM

Insert a SDRAM card compatible with your Raspberry Pi board into the card reader of your computer and follow these instructions to format it.

> Note: the contents of the card will be erased!

`fdisk` greets us with this warning.

```
$ sudo fdisk /dev/mmcblk0

Welcome to fdisk (util-linux 2.31.1).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
```

The `o` command creates a new DOS partition table on the card.

```
Command (m for help): o
Created a new DOS disklabel with disk identifier 0xc701c6c5.
```

The `n` command creates a new partition in the partition table. We just hit enter four times to accept the defaults and create a primary partition number 1 with sectors that allocate the entire space for this partition.

```
Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p): 
Partition number (1-4, default 1): 
First sector (2048-15273983, default 2048): 
Last sector, +sectors or +size{K,M,G,T,P} (2048-15273983, default 15273983): 

Created a new partition 1 of type 'Linux' and of size 7,3 GiB.
```

The partition is created with type `Linux` by default, we use the `t` command changes the type to `W95 FAT32`.

```
Command (m for help): t
Selected partition 1
Hex code (type L to list all codes): b
Changed type of partition 'Linux' to 'W95 FAT32'.
```

`p` prints the partition table so we can check if everything is ok.

```
Command (m for help): p
Disk /dev/mmcblk0: 7,3 GiB, 7820279808 bytes, 15273984 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xc701c6c5

Device         Boot Start      End  Sectors  Size Id Type
/dev/mmcblk0p1       2048 15273983 15271936  7,3G  b W95 FAT32
```

`w` then commit the changes to the card and exists `fdisk`.

```
Command (m for help): w
The partition table has been altered.
Syncing disks.

$ 
```

Before we can copy the files, we need to format the partition.

> Note: `/dev/mmcblk0p1` is the device where the first partition of *my* SDRAM card is mounted. Make sure that you use the correct device with `mkfs.fat`, as it will happily overwrite the contents of whichever device you give to it!

```
$ sudo mkfs.fat /dev/mmcblk0p1
mkfs.fat 4.1 (2017-01-24)

$
```

Now let's copy the files to the card. The card needs to have some files, which are available [here](https://github.com/raspberrypi/firmware/tree/master/boot). Download and copy these files to the root directory of the card:

* `bootcode.bin`
* `fixup.dat`
* `start.elf`

We also need to copy the kernel file, but first let's update the Makefile to build 32-bit binaries for all Raspberry Pi boards:

```
PREFIX = arm-none-eabi-
CC = $(PREFIX)gcc
CXX = $(PREFIX)g++
LD = $(PREFIX)ld
OBJDUMP = $(PREFIX)objdump
OBJCOPY = $(PREFIX)objcopy
CPPFILT = $(PREFIX)c++filt

CFLAGS   = -fsigned-char -ffreestanding -nostdinc -std=c99
INCLUDES = -I.
DEFINES  =
LDFLAGS  = -T link.T

ifeq ($(strip $(v7)),1)
  CFLAGS += -march=armv7-a -marm -mfpu=neon-vfpv4 -mfloat-abi=hard
  TARGET  = kernel7
else ifeq ($(strip $(v8)),1)
  CFLAGS += -march=armv8-a -mtune=cortex-a53 -marm -mfpu=neon-fp-armv8 -mfloat-abi=hard
  TARGET  = kernel8-32
else
  CFLAGS += -march=armv6k -mtune=arm1176jzf-s -marm -mfpu=vfp -mfloat-abi=hard
  TARGET  = kernel
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

OBJS = kmain.o memio.o gpio.o mbox.o prop.o board.o led.o

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

Now we can `make v7=1` to build `kernel7.img`, `make v8=1` to build `kernel8-32.img`, or only `make` to build `kernel.img`. Make and copy the kernel for your board to the card, insert it into the card slot, power on the board and watch the green LED blinking.

## The Timer

We're timing the blinks with a busy loop, which is of course a very bad way of doing it: different CPU with different clocks will blink the LED at different rates. The Raspberry Pi SoCs have a timer with microsecond resolution, as described in [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), chapter 12 - System Timer, so let's use it to control the LED:

```c
#ifndef TIMER_H__
#define TIMER_H__

#include <stdint.h>

uint32_t timer_high(void);
uint32_t timer_low(void);
uint64_t timer(void);

void timer_wait_until(const uint64_t t1);

#endif /* TIMER_H__ */
```

```c
#include "memio.h"

#define CS  0x00
#define CLO 0x04
#define CHI 0x08
#define C0  0x0c
#define C1  0x10
#define C2  0x14
#define C3  0x18

#define BASE_ADDR (g_baseio + UINT32_C(0x00003000))

uint32_t timer_high(void) {
  return mem_read32(BASE_ADDR + CHI);
}

uint32_t timer_low(void) {
  return mem_read32(BASE_ADDR + CLO);
}

uint64_t timer(void) {
  const uint32_t clo = BASE_ADDR + CLO;
  const uint32_t chi = BASE_ADDR + CHI;
  uint32_t high, low;

  do {
    high = mem_read32(chi);
    low = mem_read32(clo);
  }
  while (mem_read32(chi) != high);
  
  return (uint64_t)high << 32 | low;
}

void timer_wait_until(const uint64_t t1) {
  while (timer() < t1) {
    __asm volatile("");
  }
}
```

Unfortunately, this timer doesn't latch the high 32 bits when we read the low 32 bits, or vice versa. To make sure we read a consistent value, we read both values and then the high word again, repeating the process if it's different from the previous read.

Now we can change the infinite loop in `start` to use the timer and make the LED blink once per second, no matter what CPU the code is running on:

```c
  uint64_t t0 = timer();

  while (1) {
    led_set(1);
    timer_wait_until(t0 += 500000);
    led_set(0);
    timer_wait_until(t0 += 500000);
  }
```

## Source Code

This code was successfully tested in a RPi 1 Model B and a RPi 3 Model B+ and can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones05).