---
layout: post
title: How to emulate a ZX81
---

Ugh!, I started to write code for my [Auto-vectorization for the Masses](http://altdevblogaday.org/2011/05/20/auto-vectorization-for-the-masses-part-2-of-n-construction-of-the-ast/) series but I didn't manage to get it done for a new article. As I was enlighted by [this post](http://altdevblogaday.org/2011/06/12/jit-cpu-emulation-a-6502-to-x86-dynamic-recompiler-part-1/) about dynamic recompilation, I decided to write something about the subject of emulation.

Although emulating a CPU is not difficult, emulating an entire system imposes lots of challenges. In this post I'll put up a simple [ZX81](http://en.wikipedia.org/wiki/ZX81) emulator and hint the readers about the problems in getting a faithful emulation of this system.

The source code presented here is [available for download](https://github.com/leiradel/altdevblogaday/blob/master/article6/ZX81-emulator.zip?raw=true) and released under the GPL. The code is verified to compile with MinGW and a Makefile is provided.

## How the ZX81 Works

A computer is not only its CPU, there are a number of supporting chips that together make an working system you can plug to a monitor (or TV in this case) and use. Below is a list of documentation about the ZX81:

* [The ZX81 Video Display System](http://www.user.dccnet.com/wrigter/index_files/ZX%20Video%20Tutorial.htm)
* [nocash ZX81 docs](http://nocash.emubase.de/zxdocs.htm), specially the keyboard assignments
* [An Assembly Listing of the Operating System of the ZX81 ROM](http://www.wearmouth.demon.co.uk/zx81.htm)

## The CPU

Since the ZX81 is a Z80 based computer, the first thing we need is a Z80 emulator. Although writing one is a lot a fun (I've done it before), there are a lot of Z80 emulators out there we can just download and use:

* [libz80](http://libz80.sourceforge.net/)
* [Multi-Z80](http://www.koders.com/c/fid655BC2E72765665F1BA2E161F3F70D83E74C7408.aspx?s=%22Neil+Bradley%22)
* [RAZE](http://www.zophar.net/z80/raze.html)
* [Z80 Portable Emulation Package](http://fms.komkon.org/EMUL8/)
* [YAZE-AG](http://www.mathematik.uni-ulm.de/users/ag/yaze/)

YAZE-AG is the simpler to understand so it's the one we'll be using. I had to make a few changes to it to fix the IN opcode emulation and get the keyboard working. I've also done some other changes to make it easier to use.

## The ROM

The ZX81 features a built-in BASIC interpreter so we must get it in order to emulate the system. I don't know the copyright status of this ROM so I ask you to go to the Internet and find one yourself, it must have 8192 bytes. If you have a ZX81 getting dust in your garage you're entitled to use its ROM I think. I still have my old ZX81 in working condition!

Once you get the ROM, convert it to a C array we can use in our emulator. In the downloadable source code you'll find a little utility that can be used to do the conversion:

```
$ file2c zx81.rom rom > zx81rom.h
```

## First Try: Just Do It

Since we already have a Z80 emulator and the ZX81 ROM, it's understandable to be tempted to skip all the ZX81 documentation and make it run. At least that's what I did. Here's the source code to make it happen:

```c
#include <stdio.h>

#include "simz80.h"
#include "zx81rom.h"

/* address of the pointer to the beginning of the display file */
#define D_FILE 0x400c

/* the z80 state */
static struct z80 z80;

/* the memory */
static BYTE memory[ 65536 ];

/* fetches an opcode from memory */
BYTE z80_fetch( struct z80* z80, WORD a )
{
  return memory[ a ];
}

/* reads from memory */
BYTE z80_read( struct z80* z80, WORD a )
{
  return memory[ a ];
}

/* writes to memory */
void z80_write( struct z80* z80, WORD a, BYTE b )
{
  /* don't write to rom */
  if ( a >= 0x4000 )
  {
    memory[ a ] = b;
  }
}

/* reads from a port */
BYTE z80_in( struct z80* z80, WORD a )
{
  (void)z80;
  (void)a;
}

/* writes to a port */
void z80_out( struct z80* z80, WORD a, BYTE b )
{
  (void)z80;
  (void)a;
  (void)b;
}

/* setup the emulation state */
static void setup_emulation( void )
{
  memset( &z80, 0, sizeof( z80 ) );
  
  /* load rom with ghosting at 0x2000 */
  memcpy( memory + 0x0000, rom, 0x2000 );
  memcpy( memory + 0x2000, rom, 0x2000 );
  
  /* setup the registers */
  z80.pc  = 0;
  z80.iff = 0;
  z80.af_sel = z80.regs_sel = 0;
}

int main( int argc, char *argv[] )
{
  /* a counter do dump the program counter from time to time */
  int count;
  /* initialize the state */
  setup_emulation();
  
  /* emulate! */
  for ( count = 0;; count++ )
	{
    z80_step( &z80 );
    
    if ( ( count & 0xff ) == 0 )
    {
      printf( "%04x\n", z80.pc );
    }
	}

  /* we never get here... */
  return 0;
}
```

When we run this code we can see Z80's program counter (PC) wondering about the ROM so things appear to be working. But if you let it run long enough, the PC will suddenly jump to an address out of the ROM address space (from 0x0000 to 0x1fff). Changing the emulator to get the exact address gives us 0xc07d. What gives? Isn't this address in RAM? Not only it is, but it's also filled with zeroes which will make the Z80 execute NOP instructions until the 0xffff address, when the PC will go back to 0x0000 and reinitialize the system.

Well, after reading the documentation it turns out that the ZX81 uses a clever and confusing way to generate its video signal. The ZX81 has a 24x32 characters screen that is held in regular RAM, initially at the 0x407d address. As you write your BASIC program, this area, called the display file, moves towards higher RAM address to make space for your code. Worse, the video signal is generated by having the Z80 *execute* the display file with an offset of 0x8000! That's why the program counter suddenly jumps to 0xc07d (0x407d + 0x8000). The custom ULA chip works closely with the Z80 to fool it making it execute NOP and HALT instructions for timing while the ULA reads actual values from the display file and generates the video signal.

We could try to emulate the ULA and generate the emulated screen just like a real ZX81, but it quite a challenge to make it right. It involves not only the ULA, but generating maskable and non-maskable interrupts to the Z80 at the right times which requires accurate timing of the Z80 emulation. So for now, let's try to get rid of this annoyance that is the PC jumping out of the ROM and see if we can make the emulator behave.

## Second Try: Making the Emulator Behave

Reading the ZX81 ROM disassembly, we can find that the DISPLAY-5 routine is the one that makes the PC jump to the display file + 0x8000. So let's patch the ROM after loading it into the emulated memory and make DISPLAY-5 immediately return when called by poking a RET instruction at the 0x02b5 address.

```c
/* setup the emulation state */
static void setup_emulation( void )
{
  memset( &z80, 0, sizeof( z80 ) );
  
  /* load rom with ghosting at 0x2000 */
  memcpy( memory + 0x0000, rom, 0x2000 );
  memcpy( memory + 0x2000, rom, 0x2000 );
  
  /* patch DISPLAY-5 to a return */
  memory[ 0x02b5 + 0x0000 ] = 0xc9;
  memory[ 0x02b5 + 0x2000 ] = 0xc9;
  
  /* setup the registers */
  z80.pc  = 0;
  z80.iff = 0;
  z80.af_sel = z80.regs_sel = 0;
}
```

By running this version of emulator and looking at the PC values we can see that the it no longer jumps to RAM but keeps inside the ROM address space. So it looks like the emulator is working, but how we can be sure? We need some actual video display!

So lets hack a display output directly from the display file and see what we get.

## Third Try: Video Output

We'll use [SDL](http://www.libsdl.org/) to help us get the video output. It's a handy library to make portable 2D graphic applications. Here's the full listing of the emulator with the video output:

```c
#include <SDL/SDL.h>

#include "simz80.h"
#include "zx81rom.h"

/* address of the pointer to the beginning of the display file */
#define D_FILE 0x400c

/* the z80 state */
static struct z80 z80;

/* the memory */
static BYTE memory[ 65536 ];

/* the screen surface */
static SDL_Surface* screen;
/* the surface that holds the zx81 charset */
static SDL_Surface* charset;

/* fetches an opcode from memory */
BYTE z80_fetch( struct z80* z80, WORD a )
{
  return memory[ a ];
}

/* reads from memory */
BYTE z80_read( struct z80* z80, WORD a )
{
  return memory[ a ];
}

/* writes to memory */
void z80_write( struct z80* z80, WORD a, BYTE b )
{
  /* don't write to rom */
  if ( a >= 0x4000 )
  {
    memory[ a ] = b;
  }
}

/* reads from a port */
BYTE z80_in( struct z80* z80, WORD a )
{
  (void)z80;
  (void)a;
}

/* writes to a port */
void z80_out( struct z80* z80, WORD a, BYTE b )
{
  (void)z80;
  (void)a;
  (void)b;
}

/* creates a sdl surface with the zx81 character set */
static int create_charset( void )
{
  SDL_Surface* charset_rgb;
  Uint32 rmask, gmask, bmask, black, white;
  int i, addr, row, col, b;
  Uint32* pixel;
  
  /* create a rgb surface to hold 256 8x8 characters */
#if SDL_BYTEORDER == SDL_BIG_ENDIAN
  rmask = 0xff000000;
  gmask = 0x00ff0000;
  bmask = 0x0000ff00;
#else
  rmask = 0x000000ff;
  gmask = 0x0000ff00;
  bmask = 0x00ff0000;
#endif
  charset_rgb = SDL_CreateRGBSurface( SDL_SWSURFACE, 4096, 16, 32, rmask, gmask, bmask, 0 );
  
  if ( charset_rgb == NULL )
  {
    return 0;
  }
  
  /* map black and white colors */
  black = SDL_MapRGB( charset_rgb->format, 0, 0, 0 );
  white = SDL_MapRGB( charset_rgb->format, 255, 255, 255 );

  /* pixel points to the top-left pixel of the surface */
  pixel = (Uint32*)charset_rgb->pixels;
  /* addr points to the start of the characters bits */
  addr = 0x1e00;
  /* the ammount of uint32s to add to go up/down on line */
  int pitch = charset_rgb->pitch / 4;
  
  /* create the 128 characters (64 normal + 64 inverted) */
  for ( i = 0; i < 64; i++ )
  {
    for ( row = 0; row < 8; row++ )
    {
      b = rom[ addr++ ];
      
      for ( col = 0; col < 8; col++ )
      {
        if ( b & 128 )
        {
          pixel[            0 ] = black;
          pixel[            1 ] = black;
          pixel[ pitch +    0 ] = black;
          pixel[ pitch +    1 ] = black;
          
          pixel[         2048 ] = white;
          pixel[         2049 ] = white;
          pixel[ pitch + 2048 ] = white;
          pixel[ pitch + 2049 ] = white;
        }
        else
        {
          pixel[            0 ] = white;
          pixel[            1 ] = white;
          pixel[ pitch +    0 ] = white;
          pixel[ pitch +    1 ] = white;
          
          pixel[         2048 ] = black;
          pixel[         2049 ] = black;
          pixel[ pitch + 2048 ] = black;
          pixel[ pitch + 2049 ] = black;
        }
        /* advance pixel to the right */
        pixel += 2;
        b <<= 1;
      }
      /* advance pixel to the start of the next line */
      pixel += pitch * 2 - 16;
    }
    /* advance pixel to the top-left of the next character */
    pixel -= pitch * 16 - 16;
  }
  
  /* convert the rgb surface to a surface the same format of the screen */
  charset = SDL_DisplayFormat( charset_rgb );
  SDL_FreeSurface( charset_rgb );
  
  return charset != NULL;
}

/* setup the emulation state */
static void setup_emulation( void )
{
  memset( &z80, 0, sizeof( z80 ) );
  
  /* load rom with ghosting at 0x2000 */
  memcpy( memory + 0x0000, rom, 0x2000 );
  memcpy( memory + 0x2000, rom, 0x2000 );
  
  /* patch DISPLAY-5 to a return */
  memory[ 0x02b5 + 0x0000 ] = 0xc9;
  memory[ 0x02b5 + 0x2000 ] = 0xc9;
  
  /* setup the registers */
  z80.pc  = 0;
  z80.iff = 0;
  z80.af_sel = z80.regs_sel = 0;
}

static void run_some( void )
{
  int count;
  
  /*
  execute 100000 z80 instructions; the less instructions we execute here the
  slower the emulation gets, the more we execute the less responsive the
  keyboard gets
  */
  for ( count = 0; count < 100000; count++ )
  {
    z80_step( &z80 );
  }
}

static int consume_events( void )
{
  /* the event to process the window manager events */
  SDL_Event event;
  
  /* empty the event queue */
  while ( SDL_PollEvent( &event ) )
  {
    switch ( event.type )
    {
    case SDL_QUIT:
      /* quit the emulation */
      return 0;
    }
  }
  
  return 1;
}

static void update_screen( void )
{
  /* a pointer to the display file */
  WORD d_file;
  /* rects to blit from the charset to the screen */
  SDL_Rect source, dest;
  /* counters to redraw the screen */
  int row, col;
  
  /* setup invariants of the rect to address characters in the charset */
  source.y = 0;
  source.w = 16;
  source.h = 16;
  
  /* get the pointer into the display file */
  d_file = memory[ D_FILE ] | memory[ D_FILE + 1 ] << 8;
  
  /*
  redraw the screen; we could maintain a copy of the display file to avoid
  unnecessary blits
  */
  dest.y = 0;

  for ( row = 0; row < 24; row++ )
  {
    dest.x = 0;
    
    for ( col = 0; col < 32; col++ )
    {
      source.x = memory[ ++d_file ] * 16;
      SDL_BlitSurface( charset, &source, screen, &dest );
      dest.x += 16;
    }
    
    /* skip the 0x76 at the end of the line */
    d_file++;
    dest.y += 16;
  }
  
  SDL_UpdateRect( screen, 0, 0, 0, 0 );
}

int main( int argc, char *argv[] )
{
  int dont_quit;
  
  /* create our 512x384 screen; the bpp will be the same as the desktop */
  screen = SDL_SetVideoMode( 512, 384, 0, SDL_SWSURFACE );

  if ( screen == NULL )
  {
    fprintf( stderr, "Unable to set 512x384 video: %s\n", SDL_GetError() );
    return 1;
  }

  /* create the characters */
  if ( !create_charset() )
  {
    SDL_FreeSurface( screen );
    fprintf( stderr, "Unable to create charset image: %s\n", SDL_GetError() );
    return 1;
  }
  
  
  /* initialize the state */
  setup_emulation();
  
  /* emulate! */
  do
	{
    run_some();
    dont_quit = consume_events();
    update_screen();
	}
  while ( dont_quit );
  
  SDL_FreeSurface( screen );

  return 0;
}
```

The code is almost the same as the previous version. The differences are:

* We're setting up a video mode for our display output. The ZX81 resolution is 24x32 characters each having 8x8 pixels which gives us 256x192 pixels. Since a window this size on typical desktop resolutions would give us a very small window, we'll scale everything two times.
* We're creating a bitmap of the ZX81 character set directly from ROM. Not only it avoids the need of having an image with the charset around, but it also avoids problems with the charset copyright.
* We're processing the window manager's events and taking the chance to capture quit events to terminate the emulation.

So when we run it we are presented with this awesome display output:

![zx81-display-output]({{ site.url }}/assets/2011-6-11-How-to-Emulate/zx81-display-output.png)

Hooray, it works! Now only if we could interact with the emulator...

## Fourth Try: Interaction

Since we're already handling events from the window manager all we have to do is process keyboard events and hook them up into the emulation. The ZX81 reads its keyboard state via IO ports, so all we have to do is to translate keyboard events from the window manager to a format the emulator understands.

The ZX81 keyboard is divided into eight rows of five keys each, giving a total of 40 keys. When the row we want to read is output to high byte of the port address (an undocumented feature of the Z80, and one that I fixed in YAZE-AG) as a bit pattern, we must return the state of that row also as a bit pattern. For instance, outputting to port 0xfe00 we're addressing row #2, so we can return 0b11110111 to emulate the R key being held down.

The updated `z80_in`, `setup_emulation` and `consume_events` functions are listed below:

```c
/* ... */

/* the keyboard state and the memory */
static BYTE keyboard[ 9 ];
static BYTE memory[ 65536 ];

/* array to covert SDLK_* constants to row/col zx81 keyboard bits */
static BYTE sdlk2scan[ SDLK_LAST ];

/* ... */

/* reads from a port */
BYTE z80_in( struct z80* z80, WORD a )
{
  int i;
  
  /* any read where the 0th bit of the port is zero reads from the keyboard */
  if ( ( a & 1 ) == 0 )
  {
    /* get the keyboard row */
    a >>= 8;
    
    for ( i = 0; i < 8; i++ )
    {
      /* check the first zeroed bit to select the row */
      if ( ( a & 1 ) == 0 )
      {
        /* return the keyboard state for the row */
        return keyboard[ i ];
      }
      a >>= 1;
    }
  }
}

/* ... */

/* setup the emulation state */
static void setup_emulation( void )
{
  memset( &z80, 0, sizeof( z80 ) );
  
  /* load rom with ghosting at 0x2000 */
  memcpy( memory + 0x0000, rom, 0x2000 );
  memcpy( memory + 0x2000, rom, 0x2000 );
  
  /* patch DISPLAY-5 to a return */
  memory[ 0x02b5 + 0x0000 ] = 0xc9;
  memory[ 0x02b5 + 0x2000 ] = 0xc9;
  
  /* setup the registers */
  z80.pc  = 0;
  z80.iff = 0;
  z80.af_sel = z80.regs_sel = 0;
  
  /* reset the keyboard state */
  memset( keyboard, 255, sizeof( keyboard ) );
  
  /* setup the key conversion table, 8 makes unsupported keys go to limbo */
  memset( sdlk2scan, 8 << 5, sizeof( sdlk2scan ) );
  
  /*
  for each supported key, set the row on the 3 most significant bits and the
  column on the 5 least significant ones
  */
  sdlk2scan[ SDLK_LSHIFT ] = 0 << 5 |  1;
  sdlk2scan[ SDLK_RSHIFT ] = 0 << 5 |  1;
  sdlk2scan[ SDLK_z ]      = 0 << 5 |  2;
  sdlk2scan[ SDLK_x ]      = 0 << 5 |  4;
  sdlk2scan[ SDLK_c ]      = 0 << 5 |  8;
  sdlk2scan[ SDLK_v ]      = 0 << 5 | 16;
  sdlk2scan[ SDLK_a ]      = 1 << 5 |  1;
  sdlk2scan[ SDLK_s ]      = 1 << 5 |  2;
  sdlk2scan[ SDLK_d ]      = 1 << 5 |  4;
  sdlk2scan[ SDLK_f ]      = 1 << 5 |  8;
  sdlk2scan[ SDLK_g ]      = 1 << 5 | 16;
  sdlk2scan[ SDLK_q ]      = 2 << 5 |  1;
  sdlk2scan[ SDLK_w ]      = 2 << 5 |  2;
  sdlk2scan[ SDLK_e ]      = 2 << 5 |  4;
  sdlk2scan[ SDLK_r ]      = 2 << 5 |  8;
  sdlk2scan[ SDLK_t ]      = 2 << 5 | 16;
  sdlk2scan[ SDLK_1 ]      = 3 << 5 |  1;
  sdlk2scan[ SDLK_2 ]      = 3 << 5 |  2;
  sdlk2scan[ SDLK_3 ]      = 3 << 5 |  4;
  sdlk2scan[ SDLK_4 ]      = 3 << 5 |  8;
  sdlk2scan[ SDLK_5 ]      = 3 << 5 | 16;
  sdlk2scan[ SDLK_0 ]      = 4 << 5 |  1;
  sdlk2scan[ SDLK_9 ]      = 4 << 5 |  2;
  sdlk2scan[ SDLK_8 ]      = 4 << 5 |  4;
  sdlk2scan[ SDLK_7 ]      = 4 << 5 |  8;
  sdlk2scan[ SDLK_6 ]      = 4 << 5 | 16;
  sdlk2scan[ SDLK_p ]      = 5 << 5 |  1;
  sdlk2scan[ SDLK_o ]      = 5 << 5 |  2;
  sdlk2scan[ SDLK_i ]      = 5 << 5 |  4;
  sdlk2scan[ SDLK_u ]      = 5 << 5 |  8;
  sdlk2scan[ SDLK_y ]      = 5 << 5 | 16;
  sdlk2scan[ SDLK_RETURN ] = 6 << 5 |  1;
  sdlk2scan[ SDLK_l ]      = 6 << 5 |  2;
  sdlk2scan[ SDLK_k ]      = 6 << 5 |  4;
  sdlk2scan[ SDLK_j ]      = 6 << 5 |  8;
  sdlk2scan[ SDLK_h ]      = 6 << 5 | 16;
  sdlk2scan[ SDLK_SPACE ]  = 7 << 5 |  1;
  sdlk2scan[ SDLK_PERIOD ] = 7 << 5 |  2;
  sdlk2scan[ SDLK_m ]      = 7 << 5 |  4;
  sdlk2scan[ SDLK_n ]      = 7 << 5 |  8;
  sdlk2scan[ SDLK_b ]      = 7 << 5 | 16;
}

/* ... */

static int consume_events( void )
{
  /* the event to process the window manager events */
  SDL_Event event;
  /* the resulting scan of a key */
  BYTE scan;
  
  /* empty the event queue */
  while ( SDL_PollEvent( &event ) )
  {
    switch ( event.type )
    {
    case SDL_KEYDOWN:
      /* key pressed, reset the corresponding bit in the keyboard state */
      if ( event.key.keysym.sym == SDLK_BACKSPACE )
      {
        keyboard[ 0 ] &= ~1;
        keyboard[ 4 ] &= ~1;
      }
      else
      {
        scan = sdlk2scan[ event.key.keysym.sym ];
        keyboard[ scan >> 5 ] &= ~( scan & 0x1f );
      }
      break;
      
    case SDL_KEYUP:
      /* key released, set the corresponding bit in the keyboard state */
      if ( event.key.keysym.sym == SDLK_BACKSPACE )
      {
        keyboard[ 0 ] |= 1;
        keyboard[ 4 ] |= 1;
      }
      else
      {
        scan = sdlk2scan[ event.key.keysym.sym ];
        keyboard[ scan >> 5 ] |= scan & 0x1f;
      }
      break;
      
    case SDL_QUIT:
      /* quit the emulation */
      return 0;
    }
  }
  
  return 1;
}
```

If we run the emulator now we'll have an working ZX81!

![zx81-hello-world]({{ site.url }}/assets/2011-6-11-How-to-Emulate/zx81-hello-world.png)

![zx81-hello-world-run]({{ site.url }}/assets/2011-6-11-How-to-Emulate/zx81-hello-world-run.png)

## What Now?

Well, first of all read the [ZX81 Manual](http://www.zx81kit.com/online_zx81_manual.htm) and make some BASIC programs. The ZX81 keyboard layout is below, and I've included a highres version in the downloadable source code which you can use to make a poster or something. The image was rasterized from a vector drawing found [here](http://tracertong.co.uk/ttf/2011/03/zx-keyboards/).

![zx81-keyboard-small]({{ site.url }}/assets/2011-6-11-How-to-Emulate/zx81-keyboard-small.png)

After that, this is what comes to mind in order of difficulty:

* Make other keys work with the emulator, i.e. `=` and `+`
* Make load and save work. Hint: find these routines in the ZX81 ROM disassembly, patch them with RET (0xc9), catch their execution by looking at the program counter inside `run_some` and load bytes from disk to memory or save bytes from memory to disk
* Run the emulator at the correct speed instead of as fast as your computer can. This will require correct timing of the Z80 instructions
* Emulate the ULA, resulting in a faithful emulation of the display
* Static/dynamic recompile the machine code
