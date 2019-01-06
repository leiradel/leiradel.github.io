---
layout: post
title: SmartStart32.S explained
---

I'm writing bare metal code for the Rapberry Pi boards using the multicore version of [SmartStart32.S](https://github.com/LdB-ECM/Raspberry-Pi/blob/master/Multicore/SmartStart32.S) to setup the environment to start executing C code.

My intention down the road is to write my own boot loader, but for that I need to understand how this boot loader works. This post is just to assess my understanding of SmartStart and to share my findings with others.

I've changed the order of the code when it makes understanding the flow easier. I've also changed to comments to pseudo-C to make the assembly listings easier on the eyes.

## The Raspberry Pi Boot Process

The boot sequence is [as follows](https://www.raspberrypi.org/forums/viewtopic.php?f=63&t=6685):

1. The GPU is on and the ARM is off, SDRAM is also off
1. The GPU boots to its firmware located in ROM on the SoC
1. The firmware loads `bootcode.bin` to L2 cache and runs it
1. `bootcode.bin` turns on the SDRAM and loads `start.elf` into RAM
1. `start.elf` parses `config.txt` to configure some aspects of the SoC
1. `start.elf` loads `cmdline.txt`
1. `start.elf` writes some ARM code to RAM that will be executed before jumping to the kernel
1. `start.elf` loads `kernel.img` to address `0x8000` and releases the ARM CPU

SmartStart is the `kernel.img` file that is the first user-defined code that can be run.

## Entry Point

The code at `0x8000` is the entry point of the kernel. On multi-core architectures, all cores runs on system startup, but [the GPU puts code in RAM](https://github.com/dwelch67/raspberrypi/blob/master/multi00/README) that is executed before the reaching `0x8000`. This code puts cores other than 0 in a busy loop looking for values in specific memory locations to continue execution. These addresses are called Core 1/2/3 Mailbox 3 Set in [ARM Quad A7 core](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2836/QA7_rev3.4.pdf), but have nothing to do with the mailbox used to comunicate with the GPU.

Only core 0 reach the entry point, the other ones are released from the busy loop later by SmartStart. SmartStart begins by saving the pogram counter and [Current Program Status Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/I2837.html) (`CPSR`) into `r12` and `r11`, respectively, for later use.

```
  mov r12, pc         ; r12 = pc + 8
  mrs r11, cpsr       ; r11 = cpsr
  and r11, r11, #0x1F ; r11 = r11 & 0x1F
```

> In `mov r12, pc`, `r12` ends up with the value of `pc` [plus 8](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890440026.html).

The `CSPR` register can be used to get/set the CPU mode and enable/disable the IRQ and FIQ interrupts. Bit 6 disables FIQ when set, and bit 7 disables IRQ when set. Bits 4-0 are the [CPU mode](http://www.keil.com/support/man/docs/armasm/armasm_dom1359731126962.htm) (more information about these modes can be found [here](https://en.wikipedia.org/wiki/ARM_architecture#CPU_modes)). All modes are priviledged except User.

> I couldn't find the hypervisor mode value anywhere in the documentation.

> ARM CPUs are set to supervidor mode when powered on AFAIK.

> SmartStart seems to believe that both A7 and A53 boot in hypervisor mode, but I couldn't find any reference.

## Entering Supervisor Mode

SmartStart puts the core into supervisor mode if it's in hypervisor mode, taking the chance to disable both IRQ and FIQ. It does so by masking out the current CPU mode, and setting the desired one along with the IRQ and FIQ bits which will then disable these interruptions. The resulting value is put into the Saved Program Status Register (`SPSR`, which holds the value of `CPSR` prior to an exception), which will overwrite `CPSR` when the CPU switches to supervisor mode. `SPSR_cxsf` means the whole of `SPSR` will be updated: **C**ontrol, e**X**tension, **S**tatus, and **F**lags.

`msr ELR_hyp, lr` sets the link register of the hypervisor mode. In hypervisor mode, `eret` will load the program counter from `ELR_hyp` and `CPSR` from `SPSR_hyp`. Since `ELR_hyp` has the address of `.NotInHypMode` and `SPSR_hyp` has the supervisor CPU mode, the CPU will continue execution in supervisor mode at `.NotInHypMode`.

> I believe this is only needed because the exception levels of hypervisor and supervisor modes are different. When changing modes that are of the same level, it's enough to just change `CPSR` directly.

These two instructions aren't supported in ARMv6 mode, and thus are written directly in hexadecimal.

```
multicore_start:
  mrs r0, cpsr                       ; r0 = cpsr
  and r1, r0, #0x1F                  ; r1 = r0 & 0x1F
  cmp r1, #CPU_HYPMODE               ; compare r1 with 0x1A (hypervisor mode)
  bne .NotInHypMode                  ; if (r1 != 0x1A) goto .NotInHypMode

  bic r0, r0, #0x1F                  ; r0 = r0 & ~0x1F (zero the CPU mode bits)
  orr r0, r0, #CPU_SVCMODE_VALUE     ; r0 = r0 | 0x13 | 1 << 6 | 1 << 7 which
                                     ; is supervisor mode with FIQ and IRQ
                                     ; disabled
  msr spsr_cxsf, r0                  ; spsr = r0
  add lr, pc, #4                     ; lr = pc + 4, which sets lr to
                                     ; .NotInHypMode

  .long 0xE12EF30E ; msr ELR_hyp, lr ; elr_hyp = lr, where elr_hyp is the
                                     ; lr register in hypervisor mode
  .long 0xE160006E ; eret            ; pc = elr_hyp, cspr = spsr_hyp

.NotInHypMode:
```

> `cmp r1, #CPU_HYPMODE` [subtracts](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890065626.html) the contents of the register with the constant value, and updates the flags N, Z, C, and V in the `CPSR` register according to the result. The register is *not* updated with the result of the subtraction.

## Configuring the Stacks

Now that the the core is in supervisor mode with interruptions disabled, the stacks for the supervisor, FIQ, and IRQ modes are configured. Each core gets a different set of stack pointers for each of these modes. If the CPU is an ARM1176JZF-S, there's only one core and the code only has to setup one set of stacks.

To check which CPU is running the code, the value of the [Main ID Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Bgbiddeb.html) (MIDR) is read. The value is a bitfield with the following information for the CPUs we care about:

|Bits|31:24|23:20|19:16|15:4|3:0|||
|---|---|---|---|---|---|---|---|---|---|
|Meaning|Implementer|Variant|Architecture|Part no|Revision|Description|SoC|
|0x410FB767|0x41|0x0|0xF|0xB76|0x7|ARM1176JZF-S r0p7|BCM2835|
|0x410FC073|0x41|0x0|0xF|0xC07|0x3|Cortex-A7 MPCore r0p3|BCM2836|
|0x410FD034|0x41|0x0|0xF|0xD03|0x4|Cortex-A53 r0p4|BCM2837|

> Implementer 0x41 (ASCII 'A') means ARM. *Variant* and *revision* are the major and minor chip revisions. I don't know what *Architecture* is.

> I don't know if other chip revisions were used on the Raspberry Pi boards, but I think it would be better to compare only the Part no field of the register.

If the CPU is *not* an ARM1176JZF-S, the [Multiprocessor Affinity Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABHBJCI.html) (MPIDR) is used to determine which core is executing the code. Bits 1-0 are used to determine which of the four cores is running the code so the correct memory addresses are selected to setup the stacks.

> Remember that for now only core 0 in multicore architectures is running. The other cores are locked in a busy loop waiting to be released.

```
  ldr r2, =__SVC_stack_core0 ; r2 = __SVC_stack_core0
  ldr r3, =__FIQ_stack_core0 ; r3 = __FIQ_stack_core0
  ldr r4, =__IRQ_stack_core0 ; r4 = __IRQ_stack_core0
  mrc p15, 0, r0, c0, c0, 0  ; r0 = midr
  ldr r1, =#ARM6_CPU_ID      ; r1 = 0x410FB767
  cmp r1, r0                 ; compare r1 with r0 (midr with ARM1176JZF-S r0p7)
  beq set_svc_stack          ; if (r1 == r0) goto set_svc_stack
                             ; (set the stack if running on a monocore CPU)

  mrc p15, 0, r5, c0, c0, 5  ; r5 = mpidr
  ands r5, r5, #0x3          ; r5 = r5 & 3, and update the flags in cpsr
  beq set_svc_stack          ; if (r5 == 0) goto set_svc_stack
                             ; (set the stack if core 0)

  ldr r2, =__SVC_stack_core1 ; r2 = __SVC_stack_core1
  ldr r3, =__FIQ_stack_core1 ; r3 = __FIQ_stack_core1
  ldr r4, =__IRQ_stack_core1 ; r4 = __IRQ_stack_core1
  cmp r5, #1                 ; compare r5 with 1
  beq set_svc_stack          ; if (r5 == 1) goto set_svc_stack
                             ; (set the stack if core 1)

  ldr r2, =__SVC_stack_core2 ; r2 = __SVC_stack_core2
  ldr r3, =__FIQ_stack_core2 ; r3 = __FIQ_stack_core2
  ldr r4, =__IRQ_stack_core2 ; r4 = __IRQ_stack_core2
  cmp r5, #2                 ; compare r5 with 2
  beq set_svc_stack          ; if (r5 == 2) goto set_svc_stack
                             ; (set the stack if core 2)

  ldr r2, =__SVC_stack_core3 ; r2 = __SVC_stack_core3
  ldr r3, =__FIQ_stack_core3 ; r3 = __FIQ_stack_core3
  ldr r4, =__IRQ_stack_core3 ; r4 = __IRQ_stack_core3
```

> The code assumes there are either 1 or 4 cores in the CPU. If new boards with more cores need to be supported, this code will have to be updated, likely computing the stack addresses from the core numbers instead of using a series of `cmp`/`beq`.

Since the core is in supervisor mode, setting the stack for that mode is just a matter of setting `sp`. For the IRQ and FIQ modes, the core must be changed to those modes before `sp` can be changed. At the end, the CPU mode is set back to supervisor.

```
set_svc_stack:
  mov sp, r2                     ; sp = r2

  mrs r0, cpsr                   ; r0 = cpsr
  bic r0, r0, #0x1F              ; r0 = r0 & ~0x1F (zero the CPU mode bits)
  orr r0, r0, #CPU_FIQMODE_VALUE ; r0 = r0 | 0x11
  msr CPSR_c, r0                 ; cpsr = r0 (only the control field is set)
                                 ; (the core is now in FIQ mode)
  mov sp, r3                     ; sp = r3

  bic r0, r0, #0x1F              ; r0 = r0 & ~0x1F
  orr r0, r0, #CPU_IRQMODE_VALUE ; r0 = r0 | 0x12
  msr CPSR_c, r0                 ; cpsr = r0 (only the control field is set)
                                 ; (the core is now in IRQ mode)
  mov sp, r4                     ; sp = r4

  bic r0, r0, #0x1F              ; r0 = r0 & ~0x1F
  orr r0, r0, #CPU_SVCMODE_VALUE ; r0 = r0 | 0x13
  msr CPSR_c, r0                 ; cpsr = r0 (only the control field is set)
                                 ; (the core is now back to supervisor mode)
```

## Enabling the Floating Point Processor

Next, SmartStart enables the [VFP](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Chdbgjeg.html) (coprocessors 10 for single precison, and 11 for double precision) in both secure and non-secure modes by setting the corresponding bits in the [Non-Secure Access Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABGCFFJ.html) (NSACR), if not already set.

> I don't know what the differences are between secure and non-secure modes.

> I wonder if leaving coprocessor 11 off for code that only use single precision can help save energy and lower the SoC temperature. I'm not sure this is supported.

```
  mrc p15, 0, r0, c1, c1, 2 ; r0 = nsacr
  cmp r0, #0x00000C00       ; compare r0 with 0x00000C00
  beq .free_to_enable_fpu1  ; if (r0 == 0x00000C00) goto .free_to_enable_fpu1

  orr r0, r0, #0x00000C00   ; r0 = r0 | 0x00000C00
  mcr p15, 0, r0, c1, c1, 2 ; nsacr = r0

.free_to_enable_fpu1:
```

It then enable the coprocessors on both privileged and user modes by setting their bits in the [Coprocessor Access Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Bgbhaacj.html), and then enables the VFP by setting its EN flag in the [Floating-Point Exception Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Ccdhcfga.html) (FPEXC) register.

```
  mrc p15, 0, r0, c1, c0, #2         ; r0 = Coprocessor Access Control Register
  orr r0, #(0x00300000 + 0x00C00000) ; r0 = r0 | (0x00300000 + 0x00C00000)
  mcr p15, 0, r0, c1, c0, #2         ; Coprocessor Access Control Register = r0
  mov r0, #0x40000000                ; r0 = 0x40000000
  vmsr fpexc, r0                     ; fpexc = r0
```

## Enabling the L1 Cache and the Branch Predictor

After enabling VFP, SmartStart enables the L1 caches (instruction and data), and the branch predictor, by setting the respective bits in the [Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Bgbciiaf.html).

```
  mrc p15, 0, r0, c1, c0, 0 ; r0 = Control Register
  orr r0, #0x00000800       ; r0 = r0 | 0x00000800 (branch predicition)
  orr r0, #0x00000004       ; r0 = r0 | 0x00000004 (L1 data cache)
  orr r0, #0x00001000       ; r0 = r0 | 0x00001000 (L1 instruction cache)
  mcr p15, 0, r0, c1, c0, 0 ; Control Register = r0
```

> The three `orr` operations could be written as just one just like the `orr` used to modify the Coprocessor Access Control Register.

## Putting the Cores to Sleep

With the core in supervisor mode, VFP, L1 cache, and the branch predictor enabled, SmartStart puts all the cores except core 0 in wait mode. It begins by checking if the CPU is an ARM1176JZF-S or if the core is 0, in which case this code is ignored by jumping to `.cpu0_exit_multicore_park`.

> Remember that for now only core 0 is running. The other cores have *not* been released yet.

```
  mrc p15, 0, r0, c0, c0, 0     ; r0 = midr
  ldr r1, =#ARM6_CPU_ID         ; r1 = 0x410FB767
  cmp r1, r0                    ; compare r1 with r0
                                ; (midr with ARM1176JZF-S r0p7)
  beq .cpu0_exit_multicore_park ; if (r1 == r0) goto .cpu0_exit_multicore_park
                                ; (don't put the core to sleep if the CPU is
                                ; a ARM1176JZF-S)

  mrc p15, 0, r0, c0, c0, 5     ; r0 = mpidr
  ands r0, r0, #0x3             ; r0 = r0 & 3, and update the flags in cpsr
  beq .cpu0_exit_multicore_park ; if (r0 == 0) goto .cpu0_exit_multicore_park
                                ; (don't put the core to sleep if core 0)
```

For A7 and A53, SmartStart increments the number of available cores so it can be queried later, and jump to the spin loop at `SecondarySpin` that keeps the core in wait mode until made run programattically. This means cores 1 to 3 will be waiting again for a signal to run, just like they were before being released to be setup.

```
  ldr r1, =RPi_CoresReady ; r1 = &RPi_CoresReady
  ldr r0, [r1]            ; r0 = *(uint32_t*)r1
  add r0, r0, #1          ; r0 = r0 + 1
  str r0, [r1]            ; *(uint32_t*)r1 = r0
  b SecondarySpin         ; goto SecondarySpin
```

`SecondarySpin` is just a busy loop that makes the cores wait for an address to execute in the mailboxes, but uses a `wfe` instruction which [puts the core to sleep](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425917119639.html) until a `sev` instruction is executed, so the loop is not strictly busy. Note that `sev` [wakes up all cores](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425906657222.html) sleeping on `wfe`, so the run address is checked for zero so only the intended core actually runs the code, and the other ones go back to `wfe`.

When an address is written to the mailbox and the core is woken up from `wfe`, SmartStart will set `r0` to zero, `r1` to the machine ID (read from address `3138` or `0xC42`), and `r2` to the ATAG address (read from address 0x100). They are received in C land as the first three parameters to the function as per the [ARM ABI](http://infocenter.arm.com/help/topic/com.arm.doc.subset.swdev.abi/index.html).

> I don't know what this machine ID is. I didn't look into the ATAG contents yet, but [here](http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html#appendix_tag_reference)'s some explanation about them.

If the function returns, the core is put to sleep again.

```
SecondarySpin:
  mrc p15, 0, r0, c0, c0, 5        ; r0 = mpidr
  ands r0, r0, #0x3                ; r0 = r0 & 3, and update the flags in cpsr
  ldr r5, =mbox                    ; r5 = &mbox, which holds the memory address
                                   ; that contains 0x4000008C
  ldr r5, [r5]                     ; r5 = *(uint32_t*)r5, or r5 = 0x4000008C
  mov r3, #0                       ; r3 = 0
  add r5, #(0x400000CC-0x4000008C) ; r5 = r5 + 0x40

SecondarySpinLoop:
  wfe                              ; sleep waiting for an event
  ldr r4, [r5, r0, lsl #4]         ; r4 = *(uint32_t*)(r5 + r0 * 16)
                                   ; (r5 + r0 * 16 is the address of the
                                   ; mailbox for the current core)
  cmp r4, r3                       ; compare r4 with r3
                                   ; (the value read from the mailbox with 0)
  beq SecondarySpinLoop            ; if (r4 == 0) goto SecondarySpinLoop

  str r4, [r5, r0, lsl #4]         ; *(uint32_t*)(r5 + r0 * 16) = r4
  mov r0, #0                       ; r0 = 0
  ldr r1, =machid                  ; r1 = 3138
  ldr r1, [r1]                     ; r1 = *(uint32_t*)r1
  ldr r2, = atags                  ; r2 = 0x100
  ldr r2, [r2]                     ; r2 = *(uint32_t*)r2
  ldr lr, =SecondarySpin           ; lr = SecondarySpin
  bx r4                            ; goto *r4
  b SecondarySpin                  ; goto SecondarySpin
```

## Preparing the Environment

The only core in the ARM1176JZF-S case, or core 0 in the other two CPUs, continue execution by saving the values of the pogram counter and `CPSR` (saved at the very beginning) into `RPi_BootAddr` and `RPi_CPUBootMode` so they can be easily queried in C land.

```
.cpu0_exit_multicore_park:
  ldr r1, =RPi_BootAddr    ; r1 = RPi_BootAddr
  sub r12, #8              ; r12 = r12 - 8
  str r12, [r1]            ; *(uint32_t*)r1 = r12
  ldr r1, =RPi_CPUBootMode ; r1 = RPi_CPUBootMode
  str r11, [r1]            ; *(uint32_t*)r1 = r11
```

SmartStart then sets the number of cores ready to 1.

```
  mov r0, #1              ; r0 = 1
  ldr r1, =RPi_CoresReady ; r1 = RPi_CoresReady
  str r0, [r1]            ; *(uint32_t*)r1 = r0
```

SmartStart now sets the value of:

* `RPi_CPUCurrentMode` to the current CPU mode by reading `CPSR` and masking the mode bits
* `RPi_CpuId` to the value of `MIDR`
* `RPi_CompileMode` to 6 if it was compiled for ARMv6, 7 for ARMv7, and 8 for ARMv8, along with `4 << 5` meaning that it is hardcoded to support 4 cores.

```
  mrs r2, CPSR                ; r2 = cpsr
  and r2, r2, #0x1F           ; r2 = r2 & 0x1F
  ldr r1, =RPi_CPUCurrentMode ; r1 = RPi_CPUCurrentMode
  str r2, [r1]                ; *(uint32_t*)r1 = r2

  ldr r1, =RPi_CpuId          ; r1 = RPi_CpuId
  mrc p15, 0, r0, c0, c0, 0   ; r0 = midr
  str r0, [r1]                ; *(uint32_t*)r1 = r0

  eor r0, r0, r0              ; r0 = r0 ^ r0
.if (__ARM_ARCH == 6)
  mov r0, #0x06               ; r0 = 0x06, only if __ARM_ARCH == 6
.endif
.if (__ARM_ARCH == 7)
  mov r0, #0x07               ; r0 = 0x07, only if __ARM_ARCH == 7
.endif
.if (__ARM_ARCH == 8)
  mov r0, #0x08               ; r0 = 0x08, only if __ARM_ARCH == 8
.endif
  orr r0, r0, #(4 << 5)       ; r0 = r0 | 4 << 5
  ldr r1, =RPi_CompileMode    ; r1 = RPi_CompileMode
  str r0, [r1]                ; *(uint32_t*)r1 = r0
```

## Detecting the Base I/O Address

SmartStart then tries to detect the base I/O address. It does so by probing an AFAIK [undocumented AUX register](https://www.raspberrypi.org/forums/viewtopic.php?t=168898#p1085504) (see [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf) page 8 for the AUX registers documentation), which reads `0x61757830` (`"aux0"` when interpreted as ASCII characters.) The detected address is saved in `RPi_IO_Base_Addr` for use in C land. It also sets `RPi_BusAlias` which is used in the functions that translate memory addresses from ARM to VC and vice-versa.

If the detection fails, the core is halted in a busy loop in `.pi_detect_fail`.

```
  ldr r2, =#0x61757830      ; r2 = 0x61757830, "arm0"
  ldr r1, =#0x20215010      ; r1 = 0x20215010, the address of an undocumented
                            ; AUX register
  ldr r0, [r1]              ; r0 = *(uint32_t*)r1
  cmp r2, r0                ; compare r2 with r0
  bne .not_at_address_1     ; if (r2 != r0) goto .not_at_address_1
                            ; (check the other address if this one failed)

  ldr r1, =RPi_BusAlias     ; r1 = RPi_BusAlias
  mov  r0, #0x40000000      ; r0 = 0x40000000
  str  r0, [r1]             ; *(uint32_t*)r1 = r0
  ldr r1, =RPi_IO_Base_Addr ; r1 = RPi_IO_Base_Addr
  mov  r0, #0x20000000      ; r0 = 0x20000000
  str  r0, [r1]             ; *(uint32_t*)r1 = r0
  b .autodetect_done        ; goto .autodetect_done

.not_at_address_1:
  ldr r1, =#0x3f215010      ; r1 = 0x3f215010, the other possible address of
                            ; the undocumented AUX register
  ldr r0, [r1]              ; r0 = *(uint32_t*)r1
  cmp r2, r0                ; compare r2 with r0
  beq .At2ndAddress         ; if (r2 == r0) goto .At2ndAddress

.pi_detect_fail:
  b .pi_detect_fail         ; goto .pi_detect_fail
                            ; (detection failed, halt in an infinite loop)

.At2ndAddress:
  ldr r1, =RPi_BusAlias     ; r1 = RPi_BusAlias
  mov  r0, #0xC0000000      ; r0 = 0xC0000000
  str  r0, [r1]             ; *(uint32_t*)r1 = r0
  ldr r1, =RPi_IO_Base_Addr ; r1 = RPi_IO_Base_Addr
  mov  r0, #0x3f000000      ; r0 = 0x3f000000
  str  r0, [r1]             ; *(uint32_t*)r1 = r0
.autodetect_done:
```

> I believe it could set the base I/O address based on the value of MIDR.

## Setting up the Exception Vectors

To make use of IRQ and FIQ, the [exception vectors](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Babfeega.html) must be set. They also handle exceptions like undefined instructions in the instruction stream. Each entry in the vector is a 32-bit value that is an ARM32 instruction, which will be executed when entering the corresponding exception mode. It's usual to use branch instructions in the vector.

The vector is set by copying the entries from `_isr_Table` to their destination address 0, which is the default address the CPU will use for the vectors.

```
  ldr r0, =_isr_Table                         ; r0 = _isr_Table
  mov r1, #0x0000                             ; r1 = 0
  ldmia r0!, {r2, r3, r4, r5, r6, r7, r8, r9} ; r2 = *(uint32_t*)r0++
                                              ; r3 = *(uint32_t*)r0++
                                              ; r4 = *(uint32_t*)r0++
                                              ; r5 = *(uint32_t*)r0++
                                              ; r6 = *(uint32_t*)r0++
                                              ; r7 = *(uint32_t*)r0++
                                              ; r8 = *(uint32_t*)r0++
                                              ; r9 = *(uint32_t*)r0++

  stmia r1!, {r2, r3, r4, r5, r6, r7, r8, r9} ; *(uint32_t*)r1++ = r2
                                              ; *(uint32_t*)r1++ = r3
                                              ; *(uint32_t*)r1++ = r4
                                              ; *(uint32_t*)r1++ = r5
                                              ; *(uint32_t*)r1++ = r6
                                              ; *(uint32_t*)r1++ = r7
                                              ; *(uint32_t*)r1++ = r8
                                              ; *(uint32_t*)r1++ = r9

  ldmia r0!, {r2, r3, r4, r5, r6, r7, r8, r9} ; also copy the constants needed
  stmia r1!, {r2, r3, r4, r5, r6, r7, r8, r9} ; by the ldr instructions
```

The exception vector is defined as:

```
_isr_Table:
  ldr pc, _reset_h                        ; pc = *(uint32_t*)_reset_h
  ldr pc, _undefined_instruction_vector_h ; (and so forth...)
  ldr pc, _software_interrupt_vector_h
  ldr pc, _prefetch_abort_vector_h
  ldr pc, _data_abort_vector_h
  ldr pc, _unused_handler_h
  ldr pc, _interrupt_vector_h
  ldr pc, _fast_interrupt_vector_h

_reset_h:                        .word _start
_undefined_instruction_vector_h: .word hang
_software_interrupt_vector_h:    .word hang
_prefetch_abort_vector_h:        .word hang
_data_abort_vector_h:            .word hang
_unused_handler_h:               .word hang
_interrupt_vector_h:             .word irq_handler_stub
_fast_interrupt_vector_h:        .word fiq_handler_stub
```

This vector will make the CPU restart the boot loader in case of a reset exception, will jump to `irq_handler_stub` in case of an IRQ, and to `fiq_handler_stub` in case of a FIQ. All other exceptions will make the core hang in an infinite loop at `hang`.

## Zeroing the .bss

Before releasing the other cores, SmartStart zeroes the `bss` using the symbols `__bss_start__` and `__bss_end__`. These symbols are created by the linker via a special linker script used to link the kernel.

```
.bss : {
  . = ALIGN(4);
  __bss_start__ = .;
  *(.bss .bss.*)
  *(COMMON)
  __bss_end__ = .;
}
```

Script breakdown:

1. `. = ALIGN(4)`: aligns the current address to a multiple of 4
1. `__bss_start__ = .`: sets the symbol `__bss_start__` to the value of the current address
1. `*(.bss .bss.*)`: copies the contents of the `.bss` section and all sections starting with `.bss.` to the output
1. `*(COMMON)`: copies the `COMMON` section to the output
1. `__bss_end__ = .`: sets the symbol `__bss_end__` to the value of the current address

> An explanation of the `COMMON` section is available [here](https://stackoverflow.com/questions/16835716/bss-vs-common-what-goes-where).

The code that zeroes `.bss` is straighforward:

```
  ldr   r0, =__bss_start__ ; r0 = __bss_start__
  ldr   r1, =__bss_end__   ; r1 = __bss_end__
  mov   r2, #0             ; r2 = 0

.clear_bss:
  cmp   r0, r1             ; compare r0 with t1
  bge   .clear_bss_exit    ; if (r1 >= r1) goto .clear_bss_exit
                           ; (compare at the beginning to handle an empty .bss)
  str   r2, [r0]           ; *(uint32_t*)r0 = r2
  add   r0, r0, #4         ; r0 = r0 + 4
  b .clear_bss             ; goto .clear_bss

.clear_bss_exit:
```

> There are other ways to write the loop to clear the `.bss` that use less branches.

## Releasing the Cores

Now SmartStart releases the other cores and make each one of them start executing code at `multicore_start`, where they are put into supervisor mode and run the rest of the setup just like core 0, until put to sleep with `wfe` at `SecondarySpin`.

The cores are released one by one. Each core that is released ends up incrementing `RPi_CoresReady`, which is watched by core 0 in a busy loop to wait one core to be setup before releasing the next.

If the CPU is an ARM1176JZF-S, the code jumps to `.NoMultiCoreSetup` and skips the setup of the other cores. The cores setup is also skipped if `RPi_BootAddr` is *not* `0x8000`.

> I don't know why `RPi_BootAddr` is checked here.

```
  mrc p15, 0, r0, c0, c0, 0 ; r0 = midr
  ldr r1, =#ARM6_CPU_ID     ; r1 = 0x410FB767
  cmp r1, r0                ; compare r1 with r0
  beq .NoMultiCoreSetup     ; if (r1 == r0) goto .NoMultiCoreSetup

  ldr r1, =RPi_BootAddr     ; r1 = RPi_BootAddr
  ldr r0, [r1]              ; r0 = *(uint32_t*)r1
  ldr r1, =#0x8000          ; r1 = 0x8000
  cmp r1, r0                ; compare r1 with r0
  bne .NoMultiCoreSetup     ; if (r1 != r0) goto .NoMultiCoreSetup

  mov r1, #0x40000000       ; r1 = 0x40000000
  ldr r2, =multicore_start  ; r2 = multicore_start
  str r2, [r1, #0x9C]       ; *(uint32_t*)(r1 + 0x9C) = r2
                            ; (releases core 1)
  ldr r3, =RPi_CoresReady   ; r3 = RPi_CoresReady

.WaitCore1ACK:
  ldr r1, [r3]              ; r1 = *(uint32_t*)r3
  cmp r1, #2                ; compare r1 with 2
  bne .WaitCore1ACK         ; if (r1 == 2) goto .WaitCore1ACK
                            ; (busy loop waiting core 1 to get ready)

  mov r1, #0x40000000       ; r1 = 0x40000000
  ldr r2, =multicore_start  ; r2 = multicore_start
  str r2, [r1, #0xAC]       ; *(uint32_t*)(r1 + 0xAC) = r2
                            ; (releases core 2)

.WaitCore2ACK:
  ldr r1, [r3]              ; r1 = *(uint32_t*)r3
  cmp r1, #3                ; compare r1 with 3
  bne .WaitCore2ACK         ; if (r1 == 3) goto .WaitCore2ACK
                            ; (busy loop waiting core 2 to get ready)

  mov r1, #0x40000000       ; r1 = 0x40000000
  str r2, [r1, #0xBC]       ; *(uint32_t)(r1 + 0xBC) = r2
                            ; (releases core 3)

.WaitCore3ACK:
  ldr r1, [r3]              ; r1 = *(uint32_t*)r3
  cmp r1, #4                ; compare r1 with 4
  bne .WaitCore3ACK         ; if (r1 == 4) goto .WaitCore3ACK
                            ; (busy loop waiting core 3 to get ready)

.NoMultiCoreSetup:
```

## Jumping to C

After releasing the cores, SmartStart runs the `kernel_main` function that can be written in C. This function should not exit, but if it does the core will hang in an infinite loop.

```
  bl kernel_main ; lr = return address, pc = kernel_main

hang:
  b hang         ; goto hang
```

## Using Cores 1 to 3

When execution goes to C land, cores 1 to 3 are waiting on `wfe`. The function `CoreExecute` can be called from C to make one of them run an arbitrary function. Its signature is:

```c
typedef void (*CORECALLFUNC)(void);

bool CoreExecute(uint8_t coreNum, CORECALLFUNC func);
```

```
.globl CoreExecute    
.type CoreExecute, %function

CoreExecute:
  ldr r3, =RPi_CoresReady  ; r3 = RPi_CoresReady
  ldr r2, [r3]             ; r2 = *(uint32_t*)r3
  cmp r0, r2               ; compare r0 with r2 (r0 is the first argument to
                           ; CoreExecute)
  bcs CoreExecuteFail      ; if (r0 >= r3) goto CoreExecuteFail
  ldr r3, =#0x4000008C     ; r3 = 0x4000008C
  str r1, [r3, r0, lsl #4] ; *(uint32_t*)(r3 + r4 * 16) = r1 (r1 is the second
                           ; argument to CoreExecute)
  sev                      ; wake up cores (only the core for which the
                           ; function address was set will efectively run)
  mov r0, #1               ; r0 = 1 (return 1 to the caller)
  bx lr                    ; return to caller

CoreExecuteFail:
  mov r0, #0               ; r0 = 0 (return 0 to the caller)
  bx lr                    ; return to caller
```

When the core returns from the given `CORECALLFUNC` function, it will wait again in the `SecondarySpin` loop for another function to run.

## Handling Interruptions

The IRQ and FIQ handlers in assembly just prepare the environment to make a call to the C function set in the `RPi_IrqFuncAddr` variable. These handlers are called in their respective exception modes.

```
.weak irq_handler_stub
irq_handler_stub:
  sub lr, lr, #4           ; lr = lr - 4 (adjust the return address)
  srsfd sp!, #0x13         ; save lr_irq and spsp_irq onto the supervisor stack

  cps #0x13                ; enter supervisor mode
  push {r0-r3, r9, r12}    ; push the temporary register onto the stack

  mov r1, sp               ; r1 = sp
  and r1, r1, #4           ; r1 = r1 & 4
  sub sp, sp, r1           ; sp = sp - r1 (align the stack to 8 bytes)
  push {r1, lr}            ; push the alignment and the link register

  ldr r0, =RPi_IrqFuncAddr ; r0 = RPi_IrqFuncAddr
  ldr r0, [r0]             ; r0 = *(uint32_t*)r0 (get the C handler address)
  cmp r0, #0               ; compare r0 with 0
  beq no_irqset            ; if (r0 == 0) goto no_irqset
  blx r0                   ; lr = return address, pc = r0

no_irqset:  
  pop {r1, lr}             ; pop the alignment and the link register
  add sp, sp, r1           ; undo the stack alignment
  pop {r0-r3, r9, r12}     ; restore the saved temporary registers
  rfefd sp!                ; restore lr_irq and spsp_irq from the stack, and
                           ; jump to lr
```

`fiq_handler_stub` is identical to `irq_handler_stub`, except that it takes the C function hander address from `RPi_FiqFuncAddr` instead of `RPi_IrqFuncAddr`.

> The FIQ handler could take advantage of the [banked FIQ registers](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/ch02s09s01.html), `r8` to `r14`, to avoid pushing values onto the stack.

## Helper Functions

* `EnableInterrupts`: enable IRQ
* `DisableInterrupts`: disable IRQ
* `ARMaddrToGPUaddr`: translate an ARM address to GPU address
* `GPUaddrToARMaddr`: translate a GOU address to ARM address
* `core_getnum`: get the core ID, or 0 if running on a ARM1176JZF-S CPU

```
EnableInterrupts:
  cpsie i                       ; enable IRQ
  bx lr                         ; return to caller

DisableInterrupts:
  cpsid i                       ; disable IRQ
  bx lr                         ; return to caller

ARMaddrToGPUaddr:
  ldr r1, =RPi_ARM_TO_GPU_Alias ; r1 = RPi_ARM_TO_GPU_Alias
  ldr r1, [r1]                  ; r1 = *(uint32_t*)r1
  orr r0, r0, r1                ; r0 = r0 | r1 (create GPU address)
  bx  lr                        ; return to caller

GPUaddrToARMaddr:
  ldr r1, =RPi_ARM_TO_GPU_Alias ; r1 = RPi_ARM_TO_GPU_Alias
  ldr r1, [r1]                  ; r1 = *(uint32_t*)r1
  bic r0, r0, r1                ; r0 = r0 & ~r1
  bx lr                         ; return to caller

core_getnum:
  mov r0, #0                    ; r0 = 0
  mrc p15, 0, r1, c0, c0, 0     ; r1 = midr
  ldr r2, =#ARM6_CPU_ID         ; r2 = 0x410FB767
  cmp r2, r1                    ; compare r2 with r1
  beq isARM6                    ; if (r2 == r1) goto isARM6
  mrc p15, 0, r0, c0, c0, 5     ; r0 = mpidr

isARM6:
  and r0, r0, #3                ; r0 = r0 & 3
  bx lr                         ; return to caller
```

## Conclusion

Thanks for reading, this concludes the process SmartStart follows to start C code in supervisor mode in a multicore architecture. Despite some obvious opportunities for improvement, SmartStart was a great learning tool for me to understand how to setup the ARM CPU and make the transition to C land, and I hope this post helped you understand it as well.
