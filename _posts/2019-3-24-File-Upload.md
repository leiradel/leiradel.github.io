---
layout: post
title: Sending Files Via the Mini UART
---

At the end of this post we'll finally be able to transmit executables via the Mini UART, but before that we'll need to take care of a few more things.

One is the Mini UART. While polling works ok for basic usage, I had many issues while trying to receive the executable and write to the UART at the same time, in order to printf-debug my code. We'll have to use an IRQ to receive characters from the UART, so we'll see how that can be done.

Another thing are the [syscalls](https://en.wikipedia.org/wiki/System_call). We won't be separating user and kernel spaces yet, but we have to find a way to make the syscalls needed by libc arrive at their implementation in the kernel.

We'll also have to take care on how we implement the [`read` syscall](https://linux.die.net/man/2/read) for `STDIN_FILENO` and the [`write` syscall](https://linux.die.net/man/2/write) so that we can interact with a Lua REPL via the serial port. Yes, we'll run the full Lua REPL in baremetal!

So lets get started.

## Notes

### About the Code Style

I decided to use this project to experiment with immutable-by-default, after reading so much praise to Rust because of that feature. The code is thus littered with `const`s, and I'm counting on the compiler doing [return value optimization](https://en.wikipedia.org/wiki/Copy_elision#Return_value_optimization) to avoid passing output parameters as pointers. Looking at the assembly listings it's clear that this optimization works with C, regardless of being sold as a C++ optimization.

### About L1 Cache

Previously, we've enabled the caches to increase performance. However, the data cache cannot be used because of the peripherals, which use memory mapped IO. The only way to turn the data cache is to setup the MMU, and put the peripherals address range into non-cacheable pages. Since we won't be using the MMU here, we'll remove the code that turns the cache on.

> While it seems that we could at least turn on the I$, I couldn't reliably load and run programs when it's on. We'll move on with the I$ off for now.

### About Memory Barriers

All this time we've neglected an important piece of information from the Broadcom documentation. In [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), section 1.3 - Peripheral access precautions for correct memory ordering, Broadcom lays out the rules for correctly reading and writing from and to peripherals:

* A memory write barrier before the first write to a peripheral.
* A memory read barrier after the last read of a peripheral.

I've been postponing this but now I've changed the code to add those barriers where appropriate. The interesting part about this is that `memio_init` sets function pointers for the correct memory barrier implementations depending on the CPU architecture, which we then call when needed.

## Executable Format

When a kernel loads an executable into memory, a *lot* of things happen. Operating systems have the ability to run many processes simultaneously, and to increase security against attackers they also have the ability to [load the executables at randomized addresses](https://en.wikipedia.org/wiki/Address_space_layout_randomization).

We won't be supporting these things in our kernel. Our executable, differently from an [elf](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) or [Portable Executable](https://en.wikipedia.org/wiki/Portable_Executable), will be linked to be loaded always at the same starting address, `0x00010000`, and won't have any relocation or session information.

The link script for our executables is very similar to the one we've been using for the kernel:

```plain
OUTPUT_ARCH(arm)
ENTRY(mainCRTStartup)

SECTIONS {
  .text 0x00010000 : {
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
    __bss_start = .;
    *(.bss .bss.*)
    *(COMMON)
    . = ALIGN(4);
    __bss_end = .;
  }

  . = ALIGN(16);
  __heap_start = .;

  /DISCARD/ : {
    *(*)
  }
}
```

The final executable will be transformed to [IHEX](https://en.wikipedia.org/wiki/Intel_HEX), which is a good format to be transmitted over UARTs since it's made only of US-ASCII characters. It also preserves the executable entry point, which the kernel then receives and uses to execute the code.

```plain
$(TARGET).ihex: $(OBJS)
	$(LD) -o $(TARGET).elf -Map $(TARGET).map $(LDFLAGS) $(OBJS) -lc -lm -lrt
	$(OBJDUMP) -d $(TARGET).elf | $(CPPFILT) > $(TARGET).lst
	$(OBJCOPY) $(TARGET).elf -O ihex $@
	wc -c $@
```

## IRQ

When using a serial connection to interact with a program running in a different machine, it's very easy to lose characters when using polling. As an example, imagine what happens if we try to write more characters then the size of the hardware output FIFO in the UART. The CPU would would stay in a busy loop waiting for characters to be transmitted, so it could send more to the FIFO.

While the CPU is in that busy loop, characters arriving at the UART will overflow the hardware input FIFO and will be lost. Attempts to poll the input FIFO while in these busy loops proved cumbersome and not worth the trouble.

We'll solve this issue by extending the hardware FIFO with a FIFO in RAM, and using IRQ to write characters to the later as soon as they arrive in the former. By making this software FIFO big enough, we can keep the CPU in busy loops, polling the hardware output FIFO, while characters arriving will be handled correctly.

### Note About Transmitting With an IRQ

It's tempting to use the IRQ to also send characters, but this is more complicated then receiving. The reason is that the IRQ for outgoing characters only triggers when the hardware output FIFO is empty. Imagine that this is already the case, and we want to transmit one character. We can't just put it in the software FIFO, as it will sit there forever. So if the hardware FIFO is empty, we have to send it directly to the UART.

Now imagine we want to send a second character. Where we have to put it? The software FIFO is empty, but what about the hardware FIFO? Is it already empty so that we should also send this second character directly to the UART, or is it still busy and we can put the character in the sotfware FIFO to be grabbed when the IRQ triggers?

While this is not too complicated, the solution needs synchronization primitives, and I'd rather work towards other goals for now.

### User-defined Interrupt Handlers

The first thing we need is a way to change the interrupt handlers. Right now they're fixed, with the exception handlers signaling their exceptions by blinking the LED (the Green LED Of Death), and the software and hardware interrupt handlers empty. Lets fix this:

```c
#ifndef ISR_H__
#define ISR_H__

#include <stdint.h>

typedef enum {
  ISR_RESET     = 0,
  ISR_UNDEFINED = 1,
  ISR_SOFTWARE  = 2,
  ISR_PREFETCH  = 3,
  ISR_ABORT     = 4,
  ISR_IRQ       = 5,
  ISR_FIQ       = 6
}
isr_t;

typedef void (*isr_handler_t)(uint32_t const value);

isr_handler_t isr_sethandler(isr_t const isr, isr_handler_t const handler);
void isr_enablebasic(int const irq_number);
void isr_disablebasic(int const irq_number);

#endif /* ISR_H__ */
```

`isr_sethandler` overwrites the handler for the given `isr_t` with a new one, and returns the old handler. `isr_enablebasic` and `isr_disablebasic` are used to enable specific peripherals to trigger the hardware IRQ, as described in [BCM2835 ARM Peripherals](https://www.raspberrypi.org/documentation/hardware/raspberrypi/bcm2835/BCM2835-ARM-Peripherals.pdf), chapter 7 - Interrupts.

> The Basic IRQs, as they are referred to in the documentation, all map to the only hardware IRQ available in the ARM; all of them will execute the same handler, and we have to check which peripheral caused the handler to be executed.

`isr.c` is pretty straightforward, so I'll only show the implementation of the basic IRQ enable and disable:

```c
void isr_enablebasic(int const irq_number) {
  uint32_t const en1 = BASE_ADDR + 0x0210;
  uint32_t const en2 = BASE_ADDR + 0x0214;

  if (irq_number < 32) {
    mem_write32(en1, UINT32_C(1) << irq_number);
  }
  else {
    mem_write32(en2, UINT32_C(1) << (irq_number - 32));
  }
}

void isr_disablebasic(int const irq_number) {
  uint32_t const di1 = BASE_ADDR + 0x021c;
  uint32_t const di2 = BASE_ADDR + 0x0220;

  if (irq_number < 32) {
    mem_write32(di1, UINT32_C(1) << irq_number);
  }
  else {
    mem_write32(di2, UINT32_C(1) << (irq_number - 32));
  }
}
```

The functions are identical, the only difference are the registers that are used. The registers are enable-on-set, for `isr_enablebasic`, and disable-on-set, for `isr_disablebasic`. In both functions we decide which of the two register hold the bit specific to the IRQ we want to enable, and write a 1 bit to it.

### Enabling The Receive IRQ

We'll change `uart_init` to receive a pointer to a function that will be called when the IRQ triggers:

```c
void uart_init(unsigned const baudrate, uart_newchar_t const newchar) {
  // Get the clock used with the Mini UART.
  uint32_t const clock_rate = prop_getclockrate(PROP_CLOCK_CORE);

  // Set pins 14 and 15 to use with the Mini UART.
  gpio_select(14, GPIO_FUNCTION_5);
  gpio_select(15, GPIO_FUNCTION_5);

  // Turn pull up/down off for those pins.
  gpio_setpull(14, GPIO_PULL_OFF);
  gpio_setpull(15, GPIO_PULL_OFF);

  mem_dmb();

  // Enable only the Mini UART.
  mem_write32(BASE_ADDR + AUX_ENABLES, 1);
  // Disable receiving and transmitting while we configure the Mini UART.
  mem_write32(BASE_ADDR + AUX_MU_CNTL_REG, 0);
  // Turn off interrupts.
  mem_write32(BASE_ADDR + AUX_MU_IER_REG, 0);
  // Set data size to 8 bits.
  mem_write32(BASE_ADDR + AUX_MU_LCR_REG, 3);
  // Put RTS high.
  mem_write32(BASE_ADDR + AUX_MU_MCR_REG, 0);
  // Clear both receive and transmit FIFOs.
  mem_write32(BASE_ADDR + AUX_MU_IIR_REG, 0xc6);

  // Set the desired baudrate.
  uint32_t const divisor = clock_rate / (8 * baudrate) - 1;
  mem_write32(BASE_ADDR + AUX_MU_BAUD_REG, divisor);

  if (newchar != NULL) {
    // Install the IRQ handler and enable read interrupts.
    s_callback = newchar;
    isr_sethandler(ISR_IRQ, uart_read_isr);
    mem_write32(BASE_ADDR + AUX_MU_IER_REG, 5);
    isr_enablebasic(29);
  }

  // Enable receiving and transmitting.
  mem_write32(BASE_ADDR + AUX_MU_CNTL_REG, 3);
}
```

> `mem_dmb` is the memory barrier function, as discussed earlier.

When `newchar` is `NULL`, the IRQ is not enabled, and the UART must be polled for incoming characters. When it has a valid value, it's saved in a global variable, the IRQ handler is set to the UART handler, the UART is setup to trigger IRQs when there's at least one character in its input FIFO, and then the Basic IRQ 29 is enabled. From now on, `uart_read_isr` will be called whenever a new characters arrives so lets take a look at it:

```c
static uart_newchar_t s_callback;

static void uart_read_isr(uint32_t const value) {
  (void)value;
  mem_dmb();

  while (1) {
    uint32_t const iir = mem_read32(BASE_ADDR + AUX_MU_IIR_REG);

    if ((iir & 1) == 1) {
      // No Mini UART interrupt pending, return.
      break;
    }
    
    if ((iir & 6) == 4) {
      // Character available, remove it from the receive FIFO.
      uint32_t const k = mem_read32(BASE_ADDR + AUX_MU_IO_REG) & 0xff;
      // Send it to user code.
      s_callback(k);
    }
  }

  mem_dmb();
}
```

`value` is a parameter that comes from the handler in `isr.c` which calls our handler. In some cases it has useful information, i.e. the address of the undefined instruction for the UND exception handler, but here it has no meaningful value.

Memory barriers are used to make sure the correct ordering of peripheral reads and writes, and then we enter a loop to transfer all the characters from the hardware input FIFO to the software FIFO, via calls to `s_callback`.

First we check if there's a pending IRQ for the Mini UART by checking the *interrupt pending* bit of the `AUX_MU_IIR_REG` register. Since the IRQ is shared by many peripherals, we have to do this check and make sure we do have a valid character for reading in the UART.

If there's a pending interrupt, we then check the interrupt ID bits for `0b10`, which indicates that the input FIFO has at least one valid character. If there is, we read the character from `AUX_MU_IO_REG`, and send it to `s_callback`.

> Part of this code was copied from another place, and I'm not sure why this logic runs in a loop. Maybe incoming characters can accumulate in the hardware input FIFO if i.e. another interrupt comes while the current one is being served.

### The Software FIFO

The FIFO is implemented in `tty.h` and `tty.c`.

> `tty` is not a good name for this, but naming is hard.

The FIFO will only be written to by the IRQ handler, and only be read by the `read` syscall. Since we don't have multithreading (and will never have, more about this in a future post), we can implement it as a Single Producer Single Consumer [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer), which is very easy as long as we use atomic reads and writes.

```c
#ifndef TTY_H__
#define TTY_H__

#include <unistd.h>

void tty_init(void);
int  tty_canread(void);

char tty_read(void);
void tty_write(char k);

#endif /* TTY_H__ */
```

```c
#include "tty.h"
#include "aux.h"
#include "memio.h"

#include <stdint.h>

#define RX_BUFFER_SIZE 128

static uint8_t s_readbuf[RX_BUFFER_SIZE];
static uint32_t s_head, s_tail;

static uint32_t get_head(void) {
  return mem_read32((uint32_t)&s_head);
}

static void set_head(uint32_t const new_head) {
  mem_write32((uint32_t)&s_head, new_head);
}

static uint32_t get_tail(void) {
  return mem_read32((uint32_t)&s_tail);
}

static void set_tail(uint32_t const new_tail) {
  mem_write32((uint32_t)&s_tail, new_tail);
}

static void newchar(uint8_t const k) {
  // Add the character to the read ring buffer.
  uint32_t const head     = get_head();
  uint32_t const new_head = (head + 1) % RX_BUFFER_SIZE;

  if (new_head != get_tail()) {
    // There's space available for the new character.
    s_readbuf[head] = (uint8_t)k;
    set_head(new_head);
  }
}

void tty_init(void) {
  s_head = s_tail = 0;
  uart_init(115200, newchar);
}

int tty_canread(void) {
  return get_head() != get_tail();
}

char tty_read(void) {
  while (!tty_canread()) {
    // nothing
  }

  // There's data available in the receive ring buffer, return it.
  uint32_t const tail     = get_tail();
  uint8_t  const k        = s_readbuf[tail];
  uint32_t const new_tail = (tail + 1) % RX_BUFFER_SIZE;
  set_tail(new_tail);

  return (char)k;
}

void tty_write(char const k) {
  uart_write(k, 0);
}
```

The getters and setters ensure atomic reads and writes in ARM32. Reading and writing them directly doesn't ensure atomicity, as we saw in the GPIO Memory Layout section of the [part 2](https://leiradel.github.io/2019/01/28/Bss-and-Peripheral-Base.html) of this series of posts.

`newchar` is the callback that we will pass to `uart_init`, which is then called from within the interrupt handler and write a new character to the FIFO, as long as there's enough space for it. When there isn't, the characters is lost.

We zero the head and tail of the FIFO in `tty_init`, and setup the UART with the desired baudrate and the `newchar` callback. `tty_canread` shouldn't be exposed, but we have an use for it in our code that will receive executables to run via the serial connection.

`tty_read` just blocks until a character is available in the FIFO, and then removes it from the FIFO and returns it to the caller. `tty_write` just writes the character directly to the UART, blocking if the hardware output FIFO is full.

## Syscalls

The programs that we want to receive over the serial and run on the Raspberry Pi will be linked against a libc, as we saw in [the previous post](https://leiradel.github.io/2019/02/10/The-Mini-UART.html). This, or in fact any, libc will need to make syscalls for some of their functionality.

We don't make any actual distinction between kernel code and user code, but we'll pretend here that the kernel is what initializes everything, and receives and runs the programs via the serial interface, and that the user code are those programs.

Instead of using the `smc` ([Secure Monitor Call](http://infocenter.arm.com/help/topic/com.arm.doc.100076_0100_00_en/pge1425910897272.html)) instruction to make syscalls to the kernel, we'll pass to the user code a structure with pointers to them.

```c
#ifndef SYSCALLS_H__
#define SYSCALLS_H__

#include <sys/stat.h>
#include <unistd.h>
#include <sys/time.h>
#include <fcntl.h>
#include <sys/times.h>
#include <stdarg.h>

typedef struct {
  int     (*close)(int file);
  void    (*exit)(int rc);
  int     (*fstat)(int file, struct stat* pstat);
  int     (*getpid)(void);
  int     (*gettimeofday)(struct timeval* tv, void* tz);
  int     (*isatty)(int fd);
  int     (*kill)(int pid, int sig);
  int     (*link)(const char* oldpath, const char* newpath);
  off_t   (*lseek)(int file, off_t offset, int whence);
  int     (*open)(const char* filename, int flags, va_list args);
  ssize_t (*read)(int fd, void* buf, size_t count);
  clock_t (*times)(struct tms* buffer);
  int     (*unlink)(const char* pathname);
  ssize_t (*write)(int fd, const void* buf, size_t count);
}
syscalls_t;

#endif /* SYSCALLS_H__ */
```

The kernel will declare a `syscalls_t` variable, and set all of its fields to a suitable implementation of the syscall in question. Almost all syscalls in this post will just return error codes, except `read`, `write`, `exit`, and `kill`.

### `read` and `write`

These syscalls use `tty_read` and `tty_write` to read from and write to the UART.

```c
static ssize_t sys_read(int fd, void* buf, size_t count) {
  if (fd == STDIN_FILENO) {
    uint8_t* chars = (uint8_t*)buf;

    for (size_t i = 0; i < count; i++) {
      uint8_t const k = tty_read();

      if (k == 3) {
        // Ctrl-C
        sys_kill(sys_getpid(), SIGINT);
      }

      if (k == '\r') {
        *chars++ = '\n';
        tty_write('\r');
        tty_write('\n');
        break;
      }
      else if (k == '\b') {
        if (chars > (uint8_t*)buf) {
          chars--;
          tty_write('\b');
          tty_write(' ');
          tty_write('\b');
        }
      }
      else {
        *chars++ = k;
        tty_write(k);
      }
    }

    return (ssize_t)(chars - (uint8_t*)buf);
  }

  errno = EBADF;
  return -1;
}

static ssize_t sys_write(int fd, const void* buf, size_t count) {
  if (fd == STDOUT_FILENO || fd == STDERR_FILENO) {
    const uint8_t* chars = (const uint8_t*)buf;

    for (size_t i = 0; i < count; i++) {
      if (*chars == '\n') {
        tty_write('\r');
      }

      tty_write(*chars++);
    }

    return (ssize_t)count;
  }

  errno = EBADF;
  return -1;
}
```

It's not enough to just wrap the tty calls in a loop, we have to take care of some additional things to make our programs behave like they would do in a regular OS. There's something in UNIX called [line discipline](https://en.wikipedia.org/wiki/Line_discipline) (more information [here](http://www.linusakesson.net/programming/tty/)) that must be implemented in reads and writes.

So when reading we:

* Raise a `SIGINT` to terminate the program being executed when Ctrl-C is read from the UART, after writing `"^C\r\n"` to the UART.
* Write a carriage return and a line feed when Enter is read.
* Write `"\b\40\b"` when backspace is read to erase the left character, but only if we're not at the beginning of the line.

When writing, the only thing that we do is write `"\r\n"` when a line feed is written to ensure the remote terminal goes to the beginning of the next line.

While not a complete line discipline implementation, i.e. we've left out the ability of putting it in raw mode, it works for the kind of programs we'll be running for now.

Also, trying to read from anything other than the standard input will return an error, as well as trying to write to anything other than the standard output or error streams.

### `exit` and `kill`

Implementing these syscalls is important to make the kernel regain control when the program exits, either normally or abruptly i.e. via an `abort()`.

We'll implement them here using a `longjmp` to a global `jmp_buf`, which is initialized by the kernel.

```c
jmp_buf g_exitaddr;

static void sys_exit(int rc) {
  longjmp(g_exitaddr, rc);
}

static int sys_kill(int pid, int sig) {
  if (pid == sys_getpid() && sig == SIGINT) {
    longjmp(g_exitaddr, 130);
  }

  errno = EPERM;
  return -1;
}
```

> `exit(0)` is allowed and should exit the program with an exit code of `0`, but [`longjmp` will replace `0` for `1`](https://linux.die.net/man/3/longjmp) to signal `setjmp` that the return was from an explicit call to `longjmp` rather from the regular return from `setjmp ` when the `jmp_buf` is being set.

> `kill` only exits the program when the signal is `SIGINT`, in which case [the exit code is `130`](https://www.tldp.org/LDP/abs/html/exitcodes.html). While that documentation is for bash, it's trivial to see that a C program indeed terminates with `130` when Ctrl-C is pressed.

## `crt0`

Programs expect an initialized environment to be able to run. `crt0` [comprises the set of startup routines](https://en.wikipedia.org/wiki/Crt0) needed to prepare everything so that user code can run. Things like zeroing the `.bss` section and invoking static initializers are performed by `crt0`.

Our `crt0` receives the pointer to the kernel syscalls and zeroes the `.bss` before executing `main`, which is called with fake `argc`, `argv`, and `envp` arguments.

```c
#include "syscalls.h"

#include <stdint.h>
#include <string.h>

void mainCRTStartup(uint32_t const syscalls) {
  extern uint8_t __bss_start, __bss_end;
  memset(&__bss_start, 0, &__bss_end - &__bss_start);

  extern syscalls_t g_syscalls;
  g_syscalls = *(syscalls_t*)syscalls;

  extern int main(int, char**, char**);
  static char* argv[] = {"main"};
  static char* envp[] = {NULL};
  (void)main(sizeof(argv) / sizeof(argv[0]), argv, envp);
}
```

It also provides a working `sbrk` so that the program can use the heap allocation functions.

```c
// Extend the heap space at the end of the .bss section.
void* _sbrk(intptr_t increment) {
  extern char __heap_start;
  static const char* start = &__heap_start;
  const char* prev = start;

  // Keep memory returned by _sbrk 16-byte aligned.
  increment = (increment + 15) & ~15;
  start += increment;

  return (void*)prev;
}
```

The `__heap_start` symbol is provided by the linker script, after everything else in the resulting binary.

> `sbrk` should be a syscall, but it needs to know `__heap_start`. Our executable format is a flat binary without any relocation or section information like elf, so the kernel doesn't know where address to start the heap. Maybe we'll add some information to our executables in a later post.

## The Kernel

Out kernel now only needs to setup the processor, and enter a loop to read IHEX executables from the UART and run them in a loop. We already set up the processor in the previous posts, so lets see what we have to do differently here.

```c
static void undhandler(uint32_t const pc) {
  writestr("Undefined instruction at ");
  writedword(pc);
  glod(GLOD_UNDEFINED);
}

void main() {
  isr_sethandler(ISR_UNDEFINED, undhandler);

  tty_init();

  memrange_t volatile const mem = prop_armmemory();
  uint32_t volatile const sp = (mem.base + mem.size) & ~3;

  extern jmp_buf g_exitaddr;
  setjmp(g_exitaddr);

  while (1) {
    uint32_t volatile const entry = load_ihex();

    if (entry != 0) {
      writestr("Executing entry ");
      writedword(entry);
      writestr(" with sp ");
      writedword(sp);
      writestr("\r\n");

      extern syscalls_t g_syscalls;
      run_with_sp(entry, sp, (uint32_t)&g_syscalls);
    }
  }
}
```

> Accessing non-volatile local variables after a `longjmp` is undefined, since registers will have their values changed. I was indeed having random crashes that were fixed by declaring all variables in `main` volatile.

> Note that there's nothing wrong in declaring a variable `volatile` and `const` at the same time. The former will just make the compiler never cache the variable in a register, making all reads from the variable memory reads, and all writes to the variable memory writes, and the later means that we cannot change the value of the variable after its first set (but it's still written to memory once, and all access are memory reads, because it's volatile).

`main` is called indirectly by `kmain`, and it:

1. Sets up an ISR handler to catch undefined instructions, write a message, and present the GLOD
1. Initializes the `tty`
1. Gets the ARM memory range using the property interface so it can use the highest address possible for the executables' stack
1. Initializes `g_exitaddr` so that executables that call `exit` or `kill` will be terminated and cause the kernel to read another executable
1. Loads IHEX executables and runs them in a loop

> Since we're running code always in supervisor mode, there won't be any illegal accesses to memory, so there's no need to define handlers for the abort interrupts.

Previously, we were setting up the stack pointer with `setsp`. This wasn't 100% reliable so now we'll use `run_with_sp` to run a function with a given stack pointer. Functions run via `run_with_sp` only take one argument, an `uint32_t`, and also return an `uint32_t`, and must conform to the ABI.

```c
inline uint32_t run_with_sp(
  uint32_t const address, uint32_t const sp, uint32_t const args) {

  uint32_t result;

  __asm volatile(
    "mov r0, %[args]\n"
    "mov r4, sp\n"
    "mov sp, %[sp]\n"
    "blx %[address]\n"
    "mov %[result], r0\n"
    "mov sp, r4\n"
    : [result] "=r" (result)
    : [address] "r" (address)
    , [sp] "r" (sp)
    , [args] "r" (args)
    : "r0", "r4", "lr", "cc"
  );

  return result;
}
```

`r0` is set with the argument for the function that will be run. The current `sp` value is saved in the `r4` register, which is preserved between function calls, and then `sp` is set with the new stack top. Then the function is called, `sp` is restored from `r4`, and the result of the function is put into `r0` to be used by the `run_with_sp` caller.

## User Programs

Three programs were written to test this setup and make sure the kernel can reliably load and execute them without crashing, and also to make sure that we can interact with them via `read` and `write` syscalls.

1. Echo: prints characters read via `read` along with their decimal and hexadecimal codes (press `x` or Ctrl-C to exit).
1. Info: a port of the program with the same name from the previous post. It will print board information using `write`.
1. Lua: a build of stock [Lua](https://www.lua.org/) 5.3.5, which correctly runs its REPL and exits when Ctrl-C is pressed.

The most interesting of the three is Lua. There's no compatibility layer, additional code, or changes to the stock Lua code, all the code is unmodified from the source code distribution. It's linked to `crt0` and the syscalls, but this is also required when building programs to run on regular OSes.

> I had to define `LUA_32BITS` to work around an issue. Without it everything works just fine, but integer numbers (which have 64 bits without that flag) only print their upper 32 bits, i.e. `for i=1,10 do print(i) end` will print `0` 10 times, and `=1<<31` also prints `0`, but `=1<<32` prints `1`. This is very likely a wrong format string being used to turn an integer into a string, but I didn't investigate any further.

A video of the programs being uploaded and run can be watched [here](https://asciinema.org/a/234924).

## Source Code

The source code for this post can be found [here](https://github.com/leiradel/barebones-rpi/tree/master/barebones08).
