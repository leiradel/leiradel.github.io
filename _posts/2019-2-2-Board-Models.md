---
layout: post
title: Blinking the LED Part 4 - Board Models
---

## LED Pins

With the GPIO basic functionality in place, let's make that LED blink.

Different Raspberry Pi boards and/or revisions have different pin numbers for the activity LED. The best reference I could find is the [Raspberry Pi Device Tree](https://github.com/raspberrypi/firmware/blob/master/extra/dt-blob.dts). By examining the Device Tree, we can extract this information about the activity LED:

|Device|Comment|Pin|Active|
|---|---|---|---|
|pins_rev1||16|Low|
|pins_rev2||16|Low|
|pins_bplus1|Pi 1 Model B+ rev 1.1|47|High|
|pins_bplus2|Pi 1 Model B+ rev 1.2|47|High|
|pins_aplus||47|High|
|pins_2b1|Pi 2 Model B rev 1.0|47|High|
|pins_2b2|Pi 2 Model B rev 1.1|47|High|
|pins_3b1|Pi 3 Model B rev 1.0|47|High|
|pins_3b2|Pi 3 Model B rev 1.2|130|High|
|pins_3bplus||29|High|
|pins_3aplus||29|High|
|pins_cm3||NA|NA|
|pins_pi0|Pi zero|47|Low|
|pins_pi0w|Pi zero W|47|Low|
|pins_cm||NA|NA|

If we cross that information with the Raspberry Pi [Revision Codes](https://www.raspberrypi.org/documentation/hardware/raspberrypi/revision-codes/README.md), we can extract the LED pin for each different board:

|Models|Pin|Active|
|---|---|---|
|A and B|16|Low|
|Zero and Zero W|47|Low|
|3B 1.2|130|High|
|3A+ and 3B+|29|High|
|CM1 and CM3|NA|NA|
|Others|47|High|

> Unfortunately we have to assume some pins, like for the B+ revision 1.0. Let me know if it doesn't work for you.

> If you're scratching your head because the 3B 1.2 uses pin 130 and the GPIO only has 54 pins, we'll talk about it later.

So all we need is some way to identify the board model to configure the pin number and active high/low.

## Board Identification

There's a mailbox message that returns the Revision Code of the board, so we have to send this message to VideoCore, get the result, and check what board is it using the revision code.

Mailbox is the way of passing information to the VideoCore CPU embedded in the SoC. The VideoCore is actually the GPU, and it has access to board information that is not visible to the ARM CPU. Every time we need that information, we need to ask the GPU for it.

> [The Raspberry Pi SoC is in fact a GPU with some other things around it, one of them being an ARM CPU](https://lists.denx.de/pipermail/u-boot/2015-March/208201.html).

There are two mailboxes available, [Framebuffer](https://github.com/raspberrypi/firmware/wiki/Mailbox-framebuffer-interface) and [Property tags (ARM -> VC)](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface). We'll use the property tags to get the board's revision code. The [Mailbox property interface](https://github.com/raspberrypi/firmware/wiki/Mailbox-property-interface) documents the format of the messages and the tags to get information from the firmware. The mailbox offset from the peripherals base address is `0x0000b880`.

Mailboxes work by laying out values in memory which comprise a message, and passing the start memory address of the message to the VideoCore via a peripheral register. The VideoCore will parse the message and put the results, if any, back in the same message. Messages that need a return value must have enough space for the parameters they provide *and* the return value, whichever is biggest. The message must start in a 16-byte aligned address.

One issue is that the VideoCore will only understand addresses in its own bus, so we must translate addresses from ARM to it and vice versa. This bus is documented in [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), section 1.2 - Address map. The VideoCore has a 4 GB address space, but everything is allocated within the space of 1 GB and repeats three times more to fill the entire 4 GB address space. Each repetition or mirror has it's own characteristics.

* `0x00000000` to `0x3fffffff`: L1 and L2 cached
* `0x40000000` to `0x7fffffff`: L2 cache coherent
* `0x80000000` to `0xbfffffff`: L2 cached
* `0xc0000000` to `0xffffffff`: Uncached

> Note that these addresses as well as cache configuration are from the GPU perspective.

When the ARM CPU accesses the SDRAM, the GPU memory management unit (MMU) translates the ARM physical address to a GPU bus address. On BCM2835-based boards [the ARM CPU addresses are mapped to the L2 cache coherent area](https://lists.denx.de/pipermail/u-boot/2015-March/208201.html). That's because on this SoC, the ARM core doesn't have a L2 cache and it seems some tests were made and showed that it's beneficial to the overall performance if the ARM uses the GPU L2 cache.

> This MMU is the *GPU* MMU. If we turn on the ARM's own MMU, we'll have three different address spaces in flight: ARM virtual addresses (the ones that are used in the programs we write), ARM physical addresses (translated from the virtual ones by the ARM MMU), and bus addresses (either used directly by the GPU, or translated from ARM physical address to bus ones by the GPU MMU).

On BCM2836 and BCM2837-based boards, the ARM core has its own L2 cache, so in these boards ARM addresses are translated to the uncached range. We can use this information to make the address translation between these two cores:

1. ARM to GPU: `gpuaddr = armaddr | (bcm2835 ? 0x40000000 : 0xc0000000);`
1. GPU to ARM: `gpuaddr = armaddr & ~(bcm2835 ? 0x40000000 : 0xc0000000);`

Let's add the necessary code to do address translation to `memio`:

```c
#ifndef MEMIO_H__
#define MEMIO_H__

#include <stdint.h>

extern const uint32_t g_baseio;
extern const uint32_t g_busalias;

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

inline uint32_t mem_arm2vc(const uint32_t address) {
  return address | g_busalias;
}

inline uint32_t mem_vc2arm(const uint32_t address) {
  return address & ~g_busalias;
}

int memio_init(void);

#endif /* MEMIO_H__ */
```

```c
#include <stdint.h>

uint32_t g_baseio, g_busalias;

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
    // ARM physical addresses go to the L2 cache coherent bus alias.
    g_busalias = UINT32_C(0x40000000);
    return 0;
  }

  // Test for peripherals at 0x3f000000.
  if (read32(UINT32_C(0x3f215010)) == UINT32_C(0x61757830)) {
    g_baseio = UINT32_C(0x3f000000);
    // ARM physical addresses go to the uncached bus alias.
    g_busalias = UINT32_C(0xC0000000);
    return 0;
  }

  return -1;
}
```

> We're using the peripherals base address detection code to also initialize the GPU bus alias to do address translation. This works because the peripherals base is `0x20000000` on BCM2835-based boards, and `0x3f000000` on later boards.

## Message Exchange

The format used to exchange messages with the GPU is:

1. A message header:
    1. `uint32_t size`: total size of the message, in bytes
    1. `uint32_t code`: set to 0, contains a success code from the GPU when the answer arrives.
1. One or more tags
1. A message footer:
    1. `uint32_t end`: set to 0

Each tag has the following format:

1. A tag header:
    1. `uint32_t id`: the function that the GPU must execute
    1. `uint32_t size`: the size of the value buffer, in bytes
    1. `uint32_t code`: if bit 31 is 0 the value buffer contains a request, if bit 31 is 1 the value buffer contains a response and bits 30-0 contain the response length; set to 0 on a request
1. A value buffer:
    1. `uint8_t value[]`: enough space for the request or the response, whichever is bigger
1. Padding:
    1. `uint8_t padding[]`: enough bytes to pad to a 4-byte aligned address, if not already aligned

The message header and footer, and the tag header are the same for all messages, so let's create them:

```c
#ifndef MBOX_H__
#define MBOX_H__

#include <stdint.h>

typedef struct {
  uint32_t size; /* Total size of the message in bytes. */
  uint32_t code; /* Set to 0 before sending the message. */
}
mbox_msgheader_t;

typedef struct {
  uint32_t id;   /* The function to execute. */
  uint32_t size; /* The size of the value buffer in bytes. */
  uint32_t code; /* The size of the request in bytes. */
}
mbox_tagheader_t;

typedef struct {
  uint32_t end; /* Set to 0 before sending the message. */
}
mbox_msgfooter_t;

int mbox_send(void* message);

#endif /* MBOX_H__ */
```

Let's implement `mbox_send` based on the [Accessing mailbox](https://github.com/raspberrypi/firmware/wiki/Accessing-mailboxes) documentation:

```c
#include "mbox.h"
#include "memio.h"

#include <stdint.h>

#define READ0   0x00
#define PEEK0   0x10
#define SENDER0 0x14
#define STATUS0 0x18
#define CONFIG0 0x1c
#define WRITE1  0x20
#define PEEK1   0x30
#define SENDER1 0x34
#define STATUS1 0x38
#define CONFIG1 0x3c

#define TAGS 8

#define BASE_ADDR (g_baseio + UINT32_C(0x0000b880))

int mbox_send(void* msg) {
  uint32_t value;

  // Write message to mailbox.
  do {
    value = mem_read32(BASE_ADDR + STATUS1);
  }
  while ((value & UINT32_C(0x80000000)) != 0); // Mailbox full, retry.

  // Send message to channel 8: tags (ARM to VC).
  const uint32_t msgaddr = (mem_arm2vc((uint32_t)msg) & ~15) | TAGS;
  mem_write32(BASE_ADDR + WRITE1, msgaddr);

  // Wait for the response.
  do {
    do {
      value = mem_read32(BASE_ADDR + STATUS0);
    }
    while ((value & UINT32_C(0x40000000)) != 0); // Mailbox empty, retry.

    value = mem_read32(BASE_ADDR + READ0);
  }
  while ((value & 15) != TAGS); // Wrong channel, retry.

  if (((mbox_msgheader_t*)msg)->code == UINT32_C(0x80000000)) {
    return 0; // Success!
  }

  return -1; // Ugh...
}
```

Now we can read the board revision from the VideoCore.

## The *Get Board Revision* Message

The message to get the board revision has a tag number of `0x00010002`, and the following value buffer:

```c
typedef union {
  // No request.
  struct {
    uint32_t revision;
  }
  response;
}
value_t;
```

Let's write the necessary code to create the message in memory, send it, and receive back the revision code:

```c
#ifndef PROP_H__
#define PROP_H__

#include <stdint.h>

uint32_t prop_get_revision(void);

#endif /* PROP_H__ */
```

```c
#include "mbox.h"

#include <stdint.h>

uint32_t prop_get_revision(void) {
  typedef struct {
    mbox_msgheader_t header;
    mbox_tagheader_t tag;
    
    union {
      // No request.
      struct {
        uint32_t revision;
      }
      response;
    }
    value;

    mbox_msgfooter_t footer;
  }
  message_t;

  message_t msg __attribute__((aligned(16)));

  msg.header.size = sizeof(msg);
  msg.header.code = 0;
  msg.tag.id = UINT32_C(0x00010002); // Get board revision.
  msg.tag.size = sizeof(msg.value);
  msg.tag.code = 0;
  msg.footer.end = 0;

  if (mbox_send(&msg) != 0) {
    return 0;
  }

  return msg.value.response.revision;
}
```

Now we can identify the board model by using the `prop_get_revision` function, but if you looked at the Raspberry Pi revision codes, you know that there are two different formats for them, a deprecated one for the Pi 1 models, and a new one for Pi 2 and more recent models.

To make things easier, we'll write a function that decodes the revision code into a structure from which we can easily get the board model information:

```c
#ifndef BOARD_H__
#define BOARD_H__

#include <stdint.h>

typedef enum {
  BOARD_MODEL_A = 0,
  BOARD_MODEL_B = 1,
  BOARD_MODEL_A_PLUS = 2,
  BOARD_MODEL_B_PLUS = 3,
  BOARD_MODEL_2B = 4,
  BOARD_MODEL_ALPHA = 5,
  BOARD_MODEL_CM1 = 6,
  BOARD_MODEL_3B = 8,
  BOARD_MODEL_ZERO = 9,
  BOARD_MODEL_CM3 = 10,
  BOARD_MODEL_ZERO_W = 12,
  BOARD_MODEL_3B_PLUS = 13,
  BOARD_MODEL_3A_PLUS = 14
}
board_model_t;

typedef enum {
  BOARD_PROCESSOR_BCM2835 = 0,
  BOARD_PROCESSOR_BCM2836 = 1,
  BOARD_PROCESSOR_BCM2837 = 2
}
board_processor_t;

typedef enum {
  BOARD_MANUFACTURER_QISDA = -1,
  BOARD_MANUFACTURER_SONY_UK = 0,
  BOARD_MANUFACTURER_EGOMAN = 1,
  BOARD_MANUFACTURER_EMBEST = 2,
  BOARD_MANUFACTURER_SONY_JAPAN = 3,
  BOARD_MANUFACTURER_STADIUM = 5
}
board_manufacturer_t;

typedef struct {
  board_model_t model;
  board_processor_t processor;
  int rev_major;
  int rev_minor;
  unsigned ram_mb;
  board_manufacturer_t manufacturer;
}
board_t;

int board_info(board_t* board, const uint32_t revision);

#endif /* BOARD_H__ */
```

```c
#include "board.h"

#include <stddef.h>

// https://www.raspberrypi.org/documentation/hardware/raspberrypi/revision-codes/README.md
int board_info(board_t* board, const uint32_t revision) {
  static const board_t boards[] = {
    /* 0002 */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                1, 0, 256, BOARD_MANUFACTURER_EGOMAN},
    /* 0003 */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                1, 0, 256, BOARD_MANUFACTURER_EGOMAN},
    /* 0004 */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_SONY_UK},
    /* 0005 */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_QISDA},
    /* 0006 */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_EGOMAN},
    /* 0007 */ {BOARD_MODEL_A, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_EGOMAN},
    /* 0008 */ {BOARD_MODEL_A, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_SONY_UK},
    /* 0009 */ {BOARD_MODEL_A, BOARD_PROCESSOR_BCM2835,
                2, 0, 256, BOARD_MANUFACTURER_QISDA},
    /*      */ {0},
    /*      */ {0},
    /*      */ {0},
    /* 000d */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 512, BOARD_MANUFACTURER_EGOMAN},
    /* 000e */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 512, BOARD_MANUFACTURER_SONY_UK},
    /* 000f */ {BOARD_MODEL_B, BOARD_PROCESSOR_BCM2835,
                2, 0, 512, BOARD_MANUFACTURER_EGOMAN},
    /* 0010 */ {BOARD_MODEL_B_PLUS, BOARD_PROCESSOR_BCM2835,
                1, 0, 512, BOARD_MANUFACTURER_SONY_UK},
    /* 0011 */ {BOARD_MODEL_CM1, BOARD_PROCESSOR_BCM2835,
                1, 0, 512, BOARD_MANUFACTURER_SONY_UK},
    /* 0012 */ {BOARD_MODEL_A_PLUS, BOARD_PROCESSOR_BCM2835,
                1, 1, 256, BOARD_MANUFACTURER_SONY_UK},
    /* 0013 */ {BOARD_MODEL_B_PLUS, BOARD_PROCESSOR_BCM2835,
                1, 2, 512, BOARD_MANUFACTURER_EMBEST},
    /* 0014 */ {BOARD_MODEL_CM1, BOARD_PROCESSOR_BCM2835,
                1, 0, 512, BOARD_MANUFACTURER_EMBEST},
    /* 0015 */ {BOARD_MODEL_A_PLUS, BOARD_PROCESSOR_BCM2835,
                1, 1,   0, BOARD_MANUFACTURER_EMBEST},
  };

  if ((revision & UINT32_C(0x00800000)) == 0) {
    // Old style revision, lookup in the board table.
      *board = boards[revision - 2];
  }
  else {
    // New style revision, extract the fields.
    board->model = (revision >> 4) & 0xff;
    board->processor = (revision >> 12) & 0x0f;
    board->rev_major = 1;
    board->rev_minor = revision & 0x0f;
    board->ram_mb = 256 << ((revision >> 20) & 0x07);
    board->manufacturer = (revision >> 16) & 0x0f;

    if (board->manufacturer == 4) {
      board->manufacturer = BOARD_MANUFACTURER_EMBEST;
    }
  }

  return 0;
}
```

Let's not forget to update the Makefile:

```
OBJS = kmain.o memio.o gpio.o mbox.o prop.o board.o
```

With all that supporting code in place, it's almost time to blink that LED.

## Source Code

The source code listed here can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones04).
