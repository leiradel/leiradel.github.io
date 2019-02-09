---
layout: post
title: Finish Board Initialization
---

Until now, the only thing that we're initializing in our baremetal program is the `.bss` section. The only reason everything works is luck, it turns out, and if you tried to blink the LED in a Cortex-A based board maybe it actually *didn't* work.

There's still a lot to do to both have a stable environment and increase the CPU performance:

1. Enter supervisor mode (in processors that support virtualization extensions)
1. Enable the L1 instruction and data caches
1. Enable the branch predictor
1. Enable the floating-point coprocessor
1. Setup the stack for the other exception modes
1. Setup the interrupt vector

`armregs.h` has been updated to provide access to all the registers that are needed to finish setting up the ARM CPU.

## Enter Supervisor Mode

CPUs that have the virtualization extension boot into that mode. Other CPUs boot directly into supervisor mode, which is commonly used by kernels. These modes have differences, i.e. accesses to the [Coprocessor Access Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABIEBAC.html) (CPACR) have no effect while in hypervisor mode.

To enter supervisor mode, we use the same procedure as the [SmartStart32.S](https://leiradel.github.io/2019/01/06/SmartStart.html):

```c
/*
This code will put the core into the mode specified in spsr. GNU gas doesn't
support `msr ELR_hyp, r0` and `eret` in ARMv6 and ARMv7, so we use hexadecimal
constants to emit the instructions.
*/
static inline void exit_hyp(void) {
  __asm volatile(
    "add r0, pc, #4\n"
    ".long 0xe12ef300 @ msr	elr_hyp, r0\n"
    ".long 0xe160006e @ eret\n"
    :
    :
    : "r0", "cc"
  );
}

// Put the core in supervisor mode.
static void enter_svc_mode(void) {
  const uint32_t cpsr = get_cpsr();
  const uint32_t current_mode = cpsr & CPSR_M;
  const uint32_t svc_mode = (cpsr & ~CPSR_M) | CPSR_M_SVC;
  
  if (current_mode == CPSR_M_HYP) {
    // Save the banked hypervisor registers and set them back after exiting
    // this mode. Only the stack pointer is banked in hypervisor.
    const uint32_t sp = get_sp();

    // Set the SPSR value to enter supervisor mode.
    set_spsr(svc_mode);

    // Exit hypervisor mode to the mode in `svc_mode`.
    exit_hyp();

    // Set the value of the banked registers.
    set_sp(sp);
  }
  else if (current_mode != CPSR_M_SVC) {
    // For other modes it's enough to set CPSR.
    set_cpsr(svc_mode);
  }
}
```

## Enable L1 Caches

To enable the L1 caches we have to set some bits in the [System Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABJAHDA.html) (SCTLR), namely bit 2 (`C`) and bit 12 (`I`). However, this same register is used to turn branch prediction on via bit 11 (`Z`), so we turn all three in one go:

```c
/*
Turn on L1 data and instruction caches, and branch prediction flags in the
Control Register.
*/
static void enable_l1_and_branch(void) {
  const uint32_t sctlr = get_sctlr();
  set_sctlr(sctlr | SCTLR_C | SCTLR_I | SCTLR_Z);
}
```

## Enable Floating-Point

It's not hard to make gcc emit VFP instructions even for integer code. Here, `d8` is used to temporarily hold the values of `r0` and `r1`, and `d9` is used to read a 64-bit value from memory which is later put into two ARM registers with a `vmov`:

```
    8a6c:	e92d4df0 	push	{r4, r5, r6, r7, r8, sl, fp, lr}
    8a70:	e1a06000 	mov	r6, r0
    8a74:	e1a0b001 	mov	fp, r1
    8a78:	ed2d8b04 	vpush	{d8-d9}
    8a7c:	e24dd008 	sub	sp, sp, #8
    8a80:	eb000046 	bl	8ba0 <timer>
    8a84:	e59f30cc 	ldr	r3, [pc, #204]	; 8b58 <led_blink+0xec>
    8a88:	ec410b18 	vmov	d8, r0, r1
    8a8c:	ed9f9b2f 	vldr	d9, [pc, #188]	; 8b50 <led_blink+0xe4>
```

If we execute this code before VFP is enabled an undefined exception will be thrown. We were just lucky that it didn't happen, or maybe it did if you've played around with the compiler flags or have written code that uses 64-bit values.

The VFP is seen as two separate coprocessors, CP10 and CP11, which deal with single-precision and double-precision operations, respectively. We will enable them for non-secure and non-privileged modes, and then we'll enable the VFP.

> Unfortunately, [it's not possible to enable only CP10 or CP11](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABGCFFJ.html).

```c
/*
Enable CP10 and CP11 in both secure and non-secore modes, and in both
priviedged and unpriviledged modes.
*/
static void enable_vfp(void) {
  const uint32_t nsacr = get_nsacr();
  const uint32_t cp10_and_cp11_enabled = NSACR_CP10 | NSACR_CP11;

  // Enable CP10 and CP11 in both secure and non-secore modes, if necessary.
  if ((nsacr & cp10_and_cp11_enabled) != cp10_and_cp11_enabled) {
    set_nsacr(nsacr | cp10_and_cp11_enabled);
  }

  // Enable CP10 and CP11 in both priviledged and non-priviledged modes.
  const uint32_t cpacr = get_cpacr();
  set_cpacr(cpacr | CPACR_CP10 | CPACR_CP11);

  // Enable VFP.
  set_fpexc(FPEXC_EN);
}
```

## Setting Up the Interrupt Vector

Until now we've been running code in the exception mode which the CPU boots into. Here we're forcing the CPU to run in supervisor mode, but there are other exception modes we need to take care of because they're entered by the CPU to signal issues with our code.

These modes are entered via instructions placed in the [interrupt vector](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Babfeega.html). By default, this vector is at the very beginning of the address space, `0x00000000`, and it contains one instruction that is run for each of the seven different exception modes:

1. `0x00000000`: when the CPU is reset
1. `0x00000004`: when the CPU tries to execute an undefined instruction
1. `0x00000008`: when the CPU executes a `svc` ([SuperVisor Call](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425914052313.html)) instruction
1. `0x0000000c`: when a prefetched instruction enters execution but has been read from an illegal address in the current mode
1. `0x00000010`: when an instruction tries to access an illegal address in the current mode
1. `0x00000014`: unused
1. `0x00000018`: when an IRQ occurs
1. `0x0000001c`: when a FIQ occurs

> Note that undefined instruction exceptions are also triggered when the CPU finds an instruction for a coprocessor that does not exist or has not been enabled, or for instructions that cannot be executed in the current exception mode.

Each entry in the interrupt vector has space for one ARM instruction, four bytes. What we want is to make execution jump to the code that will handle the exception. ARM has a [relative branch with a reach of ±32 MB](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425889934568.html), but we'll use `ldr` ([Load with register offset](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890273534.html)) which doesn't have that limit and can branch anywhere. We just set all entries in the vector to `ldr pc, [pc, #24]`, and add words with the absolute addresses for the handlers right after the entries.

```c
// Set up the exception vector.
static void setup_isr_table(void) {
  uint32_t* isr;

  // Cheat the compiler to get a pointer to 0. Writing isr = NULL upsets the
  // compiler, and it generates completely buggy code.
  __asm volatile("mov %[pointer], #0\n" : [pointer] "=r" (isr));

  for (int i = 0; i < 8; i++) {
    *isr++ = UINT32_C(0xe59ff018); // ldr pc, [pc, #24]
  }

  extern void res_handler(void);
  extern void und_handler(void);
  extern void swi_handler(void);
  extern void pre_handler(void);
  extern void abt_handler(void);
  extern void rsr_handler(void);
  extern void irq_handler(void);
  extern void fiq_handler(void);

  *isr++ = (uint32_t)res_handler;
  *isr++ = (uint32_t)und_handler;
  *isr++ = (uint32_t)swi_handler;
  *isr++ = (uint32_t)pre_handler;
  *isr++ = (uint32_t)abt_handler;
  *isr++ = (uint32_t)rsr_handler; // unused
  *isr++ = (uint32_t)irq_handler;
  *isr   = (uint32_t)fiq_handler;
}
```

It's interesting to note that we had to use an assembly instruction to set `isr` to zero. If we write `uint32_t* isr = 0`, this is what the compiler generates:

```
00008010 <setup_isr_table>:
    8010:	e3a03000 	mov	r3, #0
    8014:	e5833000 	str	r3, [r3]
    8018:	e7f000f0 	udf	#0
```

which is obviously incorrect. The `mov` sets `isr` to zero without the compiler being aware of it, so it generates the correct code.

Lets implement the handlers:

```c
#include "glod.h"

void res_handler(void) {
  glod(GLOD_RESET);
}

void __attribute__((isr("UND"))) und_handler(void) {
  glod(GLOD_UNDEFINED);
}

void __attribute__((isr("ABORT"))) pre_handler(void) {
  glod(GLOD_PREFETCH);
}

void __attribute__((isr("ABORT"))) abt_handler(void) {
  glod(GLOD_ABORT);
}

void rsr_handler(void) {
  glod(GLOD_RESERVED);
}

void __attribute__((isr("SWI"))) swi_handler(void) {}
void __attribute__((isr("IRQ"))) irq_handler(void) {}
void __attribute__((isr("FIQ"))) fiq_handler(void) {}
```

> Note the [ARM function attributes](https://gcc.gnu.org/onlinedocs/gcc/ARM-Function-Attributes.html) used to mark these functions as handlers of specific exception levels.

`swi_handler`, `irq_handler`, and `fiq_handler` are left empty, since we won't be using interrupts or implementing user mode for now. The other handlers signal conditions in which execution cannot continue. To make it easier to report those conditions, we'll implement the Green LED Of Death, which is just making the activity LED blink a number of times repeatedly.

```c
#ifndef GLOD_H__
#define GLOD_H__

#include <stdint.h>

#ifdef GLOD_MORSE_CODE
#define GLOD_RESET     UINT32_C(0x0001599d) /* RES .-.  .  ...    */
#define GLOD_UNDEFINED UINT32_C(0x0005e7b5) /* UND ..-  -.  -..   */
#define GLOD_PREFETCH  UINT32_C(0x0006767d) /* PRE .--.  .-.  .   */
#define GLOD_ABORT     UINT32_C(0x003f95ed) /* ABO .-  -...  ---  */
#define GLOD_EXITED    UINT32_C(0x00016d79) /* EXI .  -..-  ..    */
#define GLOD_RESERVED  UINT32_C(0x00ded675) /* FUK ..-.  ..-  -.- */
#else
/* don't blink only once to avoid confusing people counting the blinks */
#define GLOD_RESET     2
#define GLOD_UNDEFINED 3
#define GLOD_PREFETCH  4
#define GLOD_ABORT     5
#define GLOD_EXITED    6
#define GLOD_RESERVED  7 /* should never happen */
#endif

void glod(const uint32_t code);

#endif /* GLOD_H__ */
```

```c
#include "glod.h"
#include "led.h"
#include "timer.h"

void glod(const uint32_t code) {
#ifdef GLOD_MORSE_CODE
  uint64_t time = timer();
  led_set(0);

  while (1) {
    uint32_t pattern = code;

    for (unsigned i = 0; i < 16; i++, pattern >>= 2) {
      const unsigned code = pattern & 3;

      switch (code) {
        case 0: /* end of sequence */
          goto out;
        
        case 2: /* space */
          timer_waituntil(time += 600000);
          break;

        case 1: /* dot */
        case 3: /* dash */
          led_set(1);
          timer_waituntil(time += code * 200000);
          led_set(0);
          timer_waituntil(time += 200000);
          break;
      }
    }
    
out: (void)0;
    timer_waituntil(time += 500000);
  }
#else
  uint64_t time = timer();

  while (1) {
    for (uint32_t i = 0; i < code; i++) {
      led_set(1);
      timer_waituntil(time += 200000);
      led_set(0);
      timer_waituntil(time += 200000);
    }

    timer_waituntil(time += 300000);
  }
#endif
}
```

## Setting Up the Stacks

The exception handlers execute in the following modes:

|Exception|Mode|
|---|---|
|Reset|Supervisor|
|Undefined Instruction|Undefined|
|Supervisor Call|Supervisor|
|Prefetch Abort|Abort|
|Data Abort|Abort|
|IRQ interrupt|IRQ|
|FIQ interrupt|FIQ|

The supervisor stack has been already taken care of: it's either set in `kmain` when the CPU boots in supervisor mode, or in `enter_svc_mode`, where it's set to be the previous hypervisor stack. That leaves us with four modes to setup stacks for: undefined, abort, IRQ, and FIQ.

We'll set one stack for both undefined and abort modes, since they signal condition where we're unable to continue execution. IRQ and FIQ will have each its own stack. These three stacks will have only 256 bytes each.

```c
// 16 KB space to setup our initial stack.
static uint64_t svc_stack[2048] __attribute__((aligned(8)));

// 256 bytes stacks for fatal exceptions.
static uint64_t glod_stack[32] __attribute__((aligned(8)));

// 256 bytes stacks for IRQ.
static uint64_t irq_stack[32] __attribute__((aligned(8)));

// 256 bytes stacks for FIQ.
static uint64_t fiq_stack[32] __attribute__((aligned(8)));

/*
Set up the stacks for all exception modes. Careful here as we're dealing with
the stack pointer.
*/
static void setup_stacks(void) {
  // Set up the exception stacks.
  const uint32_t cpsr = get_cpsr();

  const uint32_t und_mode = (cpsr & ~CPSR_M) | CPSR_M_UND;
  set_cpsr(und_mode);
  set_sp((uint32_t)((uint8_t*)glod_stack + sizeof(glod_stack)));

  const uint32_t abt_mode = (cpsr & ~CPSR_M) | CPSR_M_ABT;
  set_cpsr(abt_mode);
  set_sp((uint32_t)((uint8_t*)glod_stack + sizeof(glod_stack)));

  const uint32_t irq_mode = (cpsr & ~CPSR_M) | CPSR_M_IRQ;
  set_cpsr(irq_mode);
  set_sp((uint32_t)((uint8_t*)irq_stack + sizeof(irq_stack)));

  const uint32_t fiq_mode = (cpsr & ~CPSR_M) | CPSR_M_FIQ;
  set_cpsr(fiq_mode);
  set_sp((uint32_t)((uint8_t*)fiq_stack + sizeof(fiq_stack)));

  // Go back to the previous mode.
  set_cpsr(cpsr);
}
```

> Since we're in supervisor mode when `setup_stacks` is called, we can switch to undefined, abort, IRQ, and FIQ modes just by changing the mode bits in the [Current Program Status Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/I2837.html) (`CPSR`), because all these modes run at the same protection level.

Now if we add an `udf #0` instruction to our code, the activity LED should blink three times, pause, blink three times, pause, and so forth, signaling the CPU found an undefined instruction.

## `start`

We can now add all those initialization steps to the `start` function. Since `kmain.c` has become quite bigger, we'll also make `start` call a separate function called `main` which is where we're going to run "user" code after everything has been setup.

> We don't really run code in unprivileged, user mode, the separation we're making by having a `main` function is only logical.

```c
/*
This function is a regular C function with a  working function frame, so we
can write regular C code here without fear of touching the stack.
*/
static void __attribute__((noinline)) start(
  const uint32_t zero,
  const uint32_t machine_type,
  const uint32_t atags_addr) {

  zero_bss(zero);
  enter_svc_mode();
  enable_l1_and_branch();
  enable_vfp();
  setup_isr_table();
  setup_stacks();

  memio_init();
  led_init();

  extern void main(const uint32_t, const uint32_t);
  main(machine_type, atags_addr);

  // Do *not* return!
  glod(GLOD_EXITED);
}
```

`start` also initializes `memio` and the LED, and also signals an invalid condition if `main` exists, which it shouldn't.

## Other Initialization

We've left two things out of the initialization process: MMU setup, and CPU clock change.

The first would help us have a separation between user and kernel modes, and would make some things easier like mapping the peripherals to the same address regardless of the board model. The second would make code run faster, as some boards don't initialize the ARM clock to its maximum possible value.

Maybe we'll tackle those in the future.

## Source Code

This code was successfully tested in a RPi 1 Model B and a RPi 3 Model B+ and can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones06).
