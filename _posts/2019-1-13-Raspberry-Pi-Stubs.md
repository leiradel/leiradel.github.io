---
layout: post
title: The Raspberry Pi Stubs
---

In the [previous post](https://leiradel.github.io/2019/01/06/SmartStart.html) we dissected SmartStart32.S, an assembly boot loader that sets up the environment to run C code in bare metal.

This time we're going to dissect the boot strap code that the GPU puts on memory to start running the boot loader. This code is called *stub* in the [Raspberry repository](https://github.com/raspberrypi/tools/tree/master/armstubs), and I'm going to call it that to avoid confusion.

There are three stubs in the repository:

* `armstub.S`, for boards based on the ARM1176JZF-S CPU
* `armstub7.S`, for boards based on the Cortex-A7 MPCore CPU, and boards based on the Cortex-A53 CPU running on 32-bit mode
* `armstub8.S`, for boards based on the Cortex-A53 CPU running on 64-bit mode

Each stub works with a corresponding kernel image:

* `armstub.S`: `kernel.img`
* `armstub7.S`: `kernel7.img` and `kernel8-32.img`
* `armstub8.S`: `kernel8.img`

## The ARM Linux Boot Protocol

In the previous post, we heard of a *machine ID*. I didn't know what that was at the time, but further research to write this post taught me about the [ARM Linux Boot Protocol](https://github.com/torvalds/linux/blob/master/Documentation/arm/Booting).

ARM machines that want to load a Linux kernel must follow this protocol, which includes passing some information to it. This information is passed via the first three general-purpose ARM registers, `r0`, `r1`, and `r2`, and are set as follow:

* `r0` is set to zero
* `r1` is set to the [ARM Linux Machine Type](http://www.arm.linux.org.uk/developer/machines/)
* `r2` is set to the address of the [ATAG_CORE](http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html) tag of the [Device Tree](https://elinux.org/Device_Tree_Reference) blob

## `armstub.S`

The `armstub.S` stub is very simple.

> I've removed the [license](https://opensource.org/licenses/BSD-3-Clause) text from the listings here for the sake of brevity.

```
.section .init
.globl _start
/* the vector table */
_start:
	mov	r0, #0
	ldr	r1, machid
	ldr	r2, atags
	ldr	pc, kernel

machid:	.word 3138

.org 0xf0
.word 0x5afe570b	@ magic value to indicate firmware should overwrite atags and kernel
.word 0			@ version
atags:	.word 0x0	@ device tree address
kernel:	.word 0x0	@ kernel start address
```

The code starts at address 0 (notice the lack of an `.org` directive at the beginning).
 
Reading the assembly we can see that it sets `r0` to 0, `r1` to 3138, `r2` to 0, and `pc` to 0. The registers are set according to the protocol described earlier. 3138 is the code for [Broadcom BCM2708 Video Coprocessor](http://www.arm.linux.org.uk/developer/machines/list.php?id=3138) in the listing of ARM Linux Machines.

> SmartStart32 doesn't pass any of these values, machine ID and ATAGS address, forward to other functions.

It's interesting to notice that `r2` and `pc` aren't really set to 0. The `0x5afe570b` word (which amusingly reads *safe stob*) gives us a hint, that the firmware will overwrite these values in the stub with the actual, correct values. This is made so because the values depend on configurations in the `config.txt` [configuration file](https://elinux.org/RPiconfig), which is parsed before the stub is run.

> `0x5afe570b` is in fact a valid ARM instruction, `bpl -435148`. It's extremely unlikely however that such a small stub would need a relative branch to a location this far.

## `armstub7.S`

The 32-bit stub for the Cortex CPUs a bit more complex. Part of the code is compiled differently depending on the `BCM2710`, which is defined for the Cortex-A53 CPU running on 32-bit mode.

```
.arch_extension sec
.arch_extension virt
```

The `.arch_extension` [ARM directives](https://sourceware.org/binutils/docs/as/ARM-Directives.html) are used to add or remove architecture extensions. I couldn't find any reference, but I believe `sec` means **TrustZone security technology**, and `virt` means **Hardware virtualization support**, both availbale in the two CPUs that this code compiles for.

```
.section .init
.globl _start
/* the vector table for secure state and HYP mode */
_start:
	b jmp_loader 	/* reset */
```

The first thing the code does is jump to `jmp_loader`. This is because we're in the [exception vectors](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Babfeega.html) default area, and we need the space support from the Software Interrupt exception, the third one in the vector, to switch to non-secure mode.

```
osc:	.word 19200000
```

This word is used to set the [Counter-timer Frequency Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABJHFFD.html) (CNTFRQ), which must be programmed to the clock frequency of the system counter. This register's purpose is just to be read by anyone wanting to know the frequency of the ARM generic timer clock, and does *not* change anything in hardware. More information can be found in the ARM Architecture Reference Manual, ARMv7-A and ARMv7-R edition, chapter B8 - The Generic Timer.

> I couldn't find an authoritative source, but [anecdotal evidence](https://pinout.xyz/pinout/gpclk#) shows that the ARM generic timer is driven by a 19.2 MHz clock, which explains the value of `osc` in the stub.

```
_secure_monitor:
	movw	r1, #0x131			@ set NS, AW, FW, HVC
	mcr	p15, 0, r1, c1, c1, 0		@ write SCR (with NS bit set)
```

After `b jmp_loader` and `osc: .word 19200000`, were at address `8`, where the Software Interrupt (SWI) handler must be coded. Normally this would be a branch, but since the other positions in the vector aren't used by the stub, the handler is coded directly here.

The handler starts setting the [Secure Configuration Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABGIHHJ.html) (SCR) to `0x131` (`0b100110001`), changing the CPU as follows:

|Bit|Name|Value|Effect|
|---|---|---|---|
|0|`NS`|1|Puts the CPU in a non-secure state|
|1|`IRQ`|0|Runs the IRQ handler in the IRQ exception level|
|2|`FIQ`|0|Runs the FIQ handler in the FIQ exception level|
|3|`EA`|0|Runs the ABT handler in the ABT exception level|
|4|`FW`|1|Makes the `F` bit in [Current Program Status Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/I2837.html) (CPSR) writable in non-secure states|
|5|`AW`|1|Makes the `A` bit in `CPSR` writable in non-secure states|
|6|`nET`|0|*Not implemented*|
|7|`SCD`|0|Makes the `smc` instruction perform a Secure Monitor Call in non-secure states|
|8|`HCE`|1|Makes the `hvc` instruction perform a Hyp Call in non-secure PL1 mode|
|9|`SIF`|0|Enables secure state instruction fetches in non-secure memory|

The `F` and `A` bits in `CPSR` are used to enable and disable FIQ and ABT exceptions.

```
	movw	r0, #0x1da			@ Set HYP_MODE | F_BIT | I_BIT | A_BIT
	msr     spsr_cxfs, r0                   @ Set full SPSR
```

Continuing with the SWI handler, all fields in the Saved Program Status Register (SPSR, which holds the value of `CPSR` prior to an exception) are updated to `0x1da` (`0b111011010`). `SPSR` will then be configured as follows:

|Bit|Name|Value|Effect|
|---|---|---|---|
|4-0|`M`|0b11010|Sets the mode to HYP|
|5|`T`|0|Disables Thumb mode|
|6|`F`|1|Disables FIQ interrupts|
|7|`I`|1|Disables IRQ interrupts|
|8|`A`|1|Disables ABT interrupts|
|9|`E`|0|Little-endian mode|
|24|`J`|0|Puts the CPU in Thumb or ARM mode (ARM in our case, since `T` is set to 0)|

The other bits, `GE`, `Q`, `V`, `C`, `Z`, and `N`, are condition flags set by some operations, and are all set to 0 here.

```
	movs	pc, lr				@ return to non-secure SVC
```

`movs pc, lr` is a an exception return instruction, which will set the Program Counter to the value of the Link Register, and will copy `SPSR` to `CPSR`, causing the changes made to `SPSR` current.

```
value:	.word 0x63fff
machid:	.word 3138
mbox: 	.word 0x4000008C
```

These words are constants used elsewhere in the stub.

```
jmp_loader:
@ Check which proc we are and run proc 0 only

	mrc p15, 0, r0, c1, c0, 0 @ Read System Control Register
	orr r0, r0, #(1<<2)       @ cache enable
	orr r0, r0, #(1<<12)      @ icache enable
	mcr p15, 0, r0, c1, c0, 0 @ Write System Control Register
```

The stub starts by enabling L1 instruction and data caches by setting the corresponding bits in the [Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Bgbciiaf.html), `I` and `C` respectively.

> Interestingly, the ARM Linux Boot protocol states that, while the instruction cache may be on or off, the data cache *must* be off.

```
.if !BCM2710
	mrc p15, 0, r0, c1, c0, 1 @ Read Auxiliary Control Register
	orr r0, r0, #(1<<6)       @ SMP
	mcr p15, 0, r0, c1, c0, 1 @ Write Auxiliary Control Register
.else
	mrrc p15, 1, r0, r1, c15  @ CPU Extended Control Register
	orr r0, r0, #(1<<6)       @ SMP
	mcrr p15, 1, r0, r1, c15  @ CPU Extended Control Register
.endif
```

The stub then changes the [Auxiliary Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABGHIBG.html) (ACTLR), if compiled for the Cortex-A7. It sets the `SMP` bit of the register, which enables cache coherency.

When compiled for the Cortex-A53, the [CPU Extended Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0500j/BABGHHEJ.html) (CPUECTLR) is changed to set the `SMPEN` bit to 1, which does the same thing for this CPU.

> In both cases the documentation says cache coherency must be enabled before the caches are enabled, which is *not* what the stub does.

```
	mov r0, #1
	mcr p15, 0, r0, c14, c3, 1 @ CNTV_CTL (enable=1, imask=0)
```

The stub enables the timer by setting the `ENABLE` bit of the [Counter PL1 Virtual Timer Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABIDBIJ.html) (CNTV_CTL, see the ARM Architecture Reference Manual, ARMv7-A and ARMv7-R edition, section B4.1.31 - CNTV_CTL, Virtual Timer Control register, VMSA) to 1, and unmasks timer interrupts by setting the `IMASK` bit to 0. Notice that IRQ and FIQ are still disabled, so timer interrupts won't actually make the processor jump to the exception vectors yet.

```
@ set to non-sec
	ldr	r1, value			@ value = 0x63fff
	mcr	p15, 0, r1, c1, c1, 2		@ NSACR = all copros to non-sec
```

Permission to the [VFP](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0301h/Chdbgjeg.html) (coprocessors 10 for single precison, and 11 for double precision), as well as some other features, in non-secure mode is granted by writing `0x63fff` in the [Non-Secure Access Control Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABGCFFJ.html) (NSACR).

```
@ timer frequency
	ldr	r1, osc				@ osc = 19200000
	mcr	p15, 0, r1, c14, c0, 0		@ write CNTFRQ
```

The value of the ARM generic timer clock, 19.2 MHz, is written to `CNTFRQ`.

```
	adr	r1, _start
	mcr	p15, 0, r1, c12, c0, 1		@ set MVBAR to secure vectors
```

The stub sets the value of Monitor Vector Base Address Register (MVBAR, see the ARM Architecture Reference Manual, ARMv7-A and ARMv7-R edition, section B4.1.107 - MVBAR, Monitor Vector Base Address Register, Security Extensions.) Any exception taken in monitor mode will use the exception vectors defined at the start of the stub.

```
	mrc	p15, 0, ip, c12, c0, 0		@ save secure copy of VBAR
```

The current value of the Vector Base Address Register (VBAR, see the ARM Architecture Reference Manual, ARMv7-A and ARMv7-R edition, section B4.1.156 - VBAR, Vector Base Address Register, Security Extensions) is saved to the `ip` register ([which is the `r12` register](http://infocenter.arm.com/help/topic/com.arm.doc.dui0473m/dom1359731136117.html)) to be used later.

```
	isb
	smc	#0				@ call into MONITOR mode
```

The stub then executes a `isb` ([Instruction Synchronization Barrier](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890205000.html)), which flushes the instruction pipeline, making sure that subsequent instructions come from the cache or RAM. It's necessary because the next instruction, `smc` ([Secure Monitor Call](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425910897272.html)) will jump to the SWI handler, which location depends on the value of `MVBAR` since we're in secure mode. Since the instruction that sets `MVBAR` was just issued, `isb` makes sure it is completed before `smc` is executed.

So, `smc #0` then runs the SWI handler, which will prepare the environment and switch the processor to non-secure mode.

```
	mcr	p15, 0, ip, c12, c0, 0		@ write non-secure copy of VBAR
```

After switching to non-secure mode, the stub writes the saved value from `VBAR` in secure mode to `VBAR` in no secure mode. This register has a different value depending if the core is in secure or non-secure mode.

```
	ldr	r4, kernel			@ kernel address to execute from
```

The stub is almost jumping into the kernel, so the kernel address is put into the `r4` register.

```
	mrc     p15, 0, r0, c0, c0, 5
	ubfx    r0, r0, #0, #2
	cmp     r0, #0                          @ primary core
	beq     9f
```

The [Multiprocessor Affinity Register](http://infocenter.arm.com/help/topic/com.arm.doc.ddi0464f/BABHBJCI.html) (MPIDR) is read into the `r0` register. MPIDR has the CPU ID field, which uniquely identifies the core that is currently running the code. For both Cortex CPUs here, this is a value between 0 and 3. `ubfx` ([Unsigned Bit Field Extract](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425915375380.html)) isolates the CPU ID field, and then the stub jumps to `9` (the `f` means it's a forward jump) if this is the 0th core, which will then make it jump to the address in `r4` as we will see later. Other cores will continue running the code below.

> At first I thought the branch instruction could encode a static prediction of it being taken or not, by using `f` for forward jump, and `b` for backwards jump. Some CPUs statically predict forward branches as *not taken*, and backward branches as *taken*. The manual for the branch instruction however [doesn't mention static branch prediction](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425889934568.html).

```
	mov	r5, #1
	lsl	r5, r0
@ modify the 0xff for force_core mode
	tst	r5, #0xff                       @ enabled cores
	beq	10f
```

Next the stub calculates `1 << cpu_id` and tests it against the `0xff` mask. If the corresponding bit in the mask is 0, the code jumps to `10`, where it will park the core indefinitely, preventing it from being used afterwards. With the default `0xff` mask, all cores will continue running the code below.

```
	ldr	r5, mbox		@ mbox
	mov	r3, #0			@ magic

	add	r5, #(0x400000CC-0x4000008C)	@ mbox
```

The stub loads `mbox` (`0x4000008C`) into `r5` and adds `0x40` to it, which results in `0x400000CC`. This serves as the base address for the mailbox register (see [BCM2836 ARM-local peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2836/QA7_rev3.4.pdf), section 4.8 - Core Mailboxes) that will be used below to compute the address of the specific register to wakeup the core. `r3` is set to 0 to be used inside the park loop.

> These mailboxes aren't related to the BCM2835 ARM mailboxes 0 and 1, of which 0 is used to [talk to the VideoCore](https://github.com/raspberrypi/firmware/wiki) processor.

> In the previous post, I've referred to these mailboxes as *memory*, but they're in fact a set of memory mapped registers with different addresses to set and to read/clear them. The fact that their addresses are after `0x40000000`, bigger than the last RAM physical address for the CPU, already shows that they couldn't be just fancy names to RAM locations.

```
1:
	wfe
	ldr	r4, [r5, r0, lsl #4]
	cmp	r4, r3
	beq	1b
```

This is the park loop, where cores 1 to 3 wait for an address to run. `wfe` ([Wait For Event](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425917119639.html)) will suspend the core until an event external to the core happens. One of these events is the execution of the `sev` ([Set Event](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425906657222.html)) instruction by another core.

The `ldr` instruction computes `r4 = *(uint32_t*)(r5 + r0 * 16)`, which will put in `r4` the contents of one of the registers Core 1 Mailbox 3 Rd/Clr, Core 2 Mailbox 3 Rd/Clr, or Core 3 Mailbox 3 Rd/Clr. These registers hold the value written to the corresponding Core X Mailbox 3 Set register.

The value read from the Mailbox 3 Rd/Clr register is tested, and the core jumps back to `1` and to the `wfe` instruction if it's 0. This test must be made because the `sev` instruction will release *all* cores suspended in `wfe` when executed, so the test makes sure only the cores with an actual address to run will be release and the others will be suspended again.

```
@ clear mailbox
	str	r4, [r5, r0, lsl #4]
9:
	mov	r0, #0
	ldr	r1, machid		@ BCM2708 machine id
	ldr	r2, atags		@ ATAGS
	bx	r4
```

The core that read an value different from zero in its Mailbox 3 Rd/Clr register will then clear that register by writing anything to them in the `str` ([Store with register offset](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425890576693.html)) instruction. It will then initialize the registers as per the ARM Linux Boot Protocol, and jump to `r4`.

> Core 0 has jumped to `9` long ago, with the kernel address in `r4`, making it run the kernel with the expected values in the registers as per the boot protocol.

> A write to the Mailbox-clear registers will set its bits to 0, but only where the value written has its bits *set to 1*. Since the code writes to this registers the value that was just read from the corresponding read register, the result is that the mailbox is set to 0. Details about how these registers work can be found in the BCM2836 ARM-local peripherals, section 4.1 - Write-set / Write-clear registers.

> The instruction used to jump to the address in the mailbox register is `bx` ([Branch and Exchange](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425889997181.html)), which will *not* set `lr`, the Link Register, with the return address. This means execution is not supposed to return here, and that each core will be parked in the stub only once and then are supposed to run user code forever unless parked again also in user code. SmartStart.S does exactly that in its SecondarySpin loop.

```
10:
	wfi
	b	10b
```

This is where disabled cores are directed to. The `wfi` ([Wait for Interrupt](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425917144829.html)) instruction works very similar to `wfe`, but does *not* wake up when `sev` is executed. Along with the branch back to `10`, this will effectively make disabled cores stay suspended forever in the stub.

> I couldn't find anything about the purpose of disabling cores in the stub. I believe it's unlike that this value is tweaked by the GPU (it would be after the `0x5afe570b` magic value to be easily located if it was the case), so it's likely that none of them will ever make it to `wfi`.

```
.org 0xf0
.word 0x5afe570b	@ magic value to indicate firmware should overwrite atags and kernel
.word 0			@ version
atags:	.word 0x0	@ device tree address
kernel:	.word 0x0	@ kernel start address
```

The last piece of the stub are just the variables used elsewhere in the code that can be tweaked by the GPU, just like in the `armstub.S` stub.

## `armstub8.S`

```
#define BIT(x) (1 << (x))

#define LOCAL_CONTROL		0x40000000
#define LOCAL_PRESCALER		0x40000008

#define OSC_FREQ		19200000

#define SCR_RW			BIT(10)
#define SCR_HCE			BIT(8)
#define SCR_SMD			BIT(7)
#define SCR_RES1_5		BIT(5)
#define SCR_RES1_4		BIT(4)
#define SCR_NS			BIT(0)
#define SCR_VAL \
    (SCR_RW | SCR_HCE | SCR_SMD | SCR_RES1_5 | SCR_RES1_4 | SCR_NS)

#define CPUECTLR_EL1		S3_1_C15_C2_1
#define CPUECTLR_EL1_SMPEN	BIT(6)

#define SPSR_EL3_D		BIT(9)
#define SPSR_EL3_A		BIT(8)
#define SPSR_EL3_I		BIT(7)
#define SPSR_EL3_F		BIT(6)
#define SPSR_EL3_MODE_EL2H	9
#define SPSR_EL3_VAL \
    (SPSR_EL3_D | SPSR_EL3_A | SPSR_EL3_I | SPSR_EL3_F | SPSR_EL3_MODE_EL2H)

.globl _start
_start:
	/*
	 * LOCAL_CONTROL:
	 * Bit 9 clear: Increment by 1 (vs. 2).
	 * Bit 8 clear: Timer source is 19.2MHz crystal (vs. APB).
	 */
	mov x0, LOCAL_CONTROL
	str wzr, [x0]
	/* LOCAL_PRESCALER; divide-by (0x80000000 / register_val) == 1 */
	mov w1, 0x80000000
	str w1, [x0, #(LOCAL_PRESCALER - LOCAL_CONTROL)]

	/* Set up CNTFRQ_EL0 */
	ldr x0, =OSC_FREQ
	msr CNTFRQ_EL0, x0

	/* Set up CNTVOFF_EL2 */
	msr CNTVOFF_EL2, xzr

	/* Enable FP/SIMD */
	/* All set bits below are res1; bit 10 (TFP) is set to 0 */
	mov x0, #0x33ff
	msr CPTR_EL3, x0

	/* Set up SCR */
	mov x0, #SCR_VAL
	msr SCR_EL3, x0

	/* Set SMPEN */
	mov x0, #CPUECTLR_EL1_SMPEN
	msr CPUECTLR_EL1, x0

	/*
	 * Set up SCTLR_EL2
	 * All set bits below are res1. LE, no WXN/I/SA/C/A/M
	 */
	ldr x0, =0x30c50830
	msr SCTLR_EL2, x0

	/* Switch to EL2 */
	mov x0, #SPSR_EL3_VAL
	msr spsr_el3, x0
	adr x0, in_el2
	msr elr_el3, x0
	eret
in_el2:

	mrs x6, MPIDR_EL1
	and x6, x6, #0x3
	cbz x6, primary_cpu

	adr x5, spin_cpu0
secondary_spin:
	wfe
	ldr x4, [x5, x6, lsl #3]
	cbz x4, secondary_spin
	mov x0, #0
	b boot_kernel

primary_cpu:
	ldr w4, kernel_entry32
	ldr w0, dtb_ptr32

boot_kernel:
	mov x1, #0
	mov x2, #0
	mov x3, #0
	br x4

.ltorg

.org 0xd8
.globl spin_cpu0
spin_cpu0:
	.quad 0
.org 0xe0
.globl spin_cpu1
spin_cpu1:
	.quad 0
.org 0xe8
.globl spin_cpu2
spin_cpu2:
	.quad 0
.org 0xf0
.globl spin_cpu3
spin_cpu3:
	# Shared with next two symbols/.word
	# FW clears the next 8 bytes after reading the initial value, leaving
	# the location suitable for use as spin_cpu3
.org 0xf0
.globl stub_magic
stub_magic:
	.word 0x5afe570b
.org 0xf4
.globl stub_version
stub_version:
	.word 0
.org 0xf8
.globl dtb_ptr32
dtb_ptr32:
	.word 0x0
.org 0xfc
.globl kernel_entry32
kernel_entry32:
	.word 0x0

.org 0x100
.globl dtb_space
dtb_space:
```
