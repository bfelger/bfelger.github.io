---
layout: default
title:  "Resurrecting ROM Pt. 2 - Getting it to Compile With Clang and Cygwin"
date:   2023-05-09 20:00:00 -0500
categories: mud C clang cygwin
---
# Resurrecting ROM Pt. 2 &mdash; Getting it to Compile With Clang and Cygwin

_Continued studies in grave-robbing_

This post picks up where [Part 1](pt-1-compile-gcc) left off. [Here is the code](https://github.com/bfelger/Mud98/tree/51a10fee23516ad44250bc652bae78f52460d942) that resulted from that work. So far, I have ROM building in GCC on Ubuntu without errors or warnings.

## Compiling for Clang 14

The basic command line arguments for Clang are similar to GCC for the sake of acting as a drop in replacement. So I will create a new `Makefile.clang` by copying the existing `Makefile`:

```
cp Makefile Makefile.clang
```

And then simply swap out `gcc` for `clang`:

```makefile
CC      = clang
```

The method for building is largely the same, except that I specify the makefile I want:

```
make clean && make -f Makefile.clang
```

That's not bad, at all. Only one warning. I'll take care of it in short order.

### A Null String, or String Null?

Here's the one warning we get, in `colour()` in `comm.c`. This is actually a fun warning that demonstrates the danger of drawing correlations from seemingly similar syntax. Today we learn that operators mean different things in different contexts:

```
comm.c:2445:15: warning: expression which evaluates to zero treated as a null pointer constant of type 'char *' [-Wnon-literal-null-conversion]
    char* p = '\0';
              ^~~~
```

The offending code is in plain view, here. Can you tell what wrong with it? It's a bit obtuse, but let's look at how it's actually used later to glean more clues:

```c
p = code;
while (*p != '\0') {
    *string = *p++;
    *++string = '\0';
}
```

This function is called by `send_to_char()`, which sends data to player sockets. It chunks up the message to send, using the open-curly-brace character (`{`) as a cue to set the VT100 color code (used by telnet). `colour()` parses the next character, emits the corresponding VT100 color code, and then emits the rest of the chunk straight to the buffer. That last part is what is happening, here. It iterates one character at a time, copying it to the input chunk to the output buffer. It doesn't use indices, as they will mismatch: ColoUr codes are single character, but their corresponding VT100 codes are multi-byte. The resultant string in the output buffer is longer than the input string that produced it.

But how does this relate to the warning? Take a look at the boolean condition:

```C
*p != '\0'
```

This says, "if the first character at `p` is _not_ `0x00`"; so when it hits the null-terminator, it stops. That's all well and good. Let's take another look at the initializer for `p`:

```c
char* p = '\0';
```

This looks correct, doesn't it? It's establishing a "false" initial value for the condition we will eventually test. This is fairly standard procedure, but in this case, it was done incorrectly. It becomes more obvious when we go back and look at what the _actual_ original code was, before we ran it through `clang-format`:

```c
char	*p = '\0';
```

They tried to initialize a pointer to `p`, which they mistook for a pointer-dereference. This is because they took the _wrong side_ in a holy war since the 1970's:

> THOU SHALT NOT MAKE POINTER DECLARATIONS LOOK LIKE POINTER DEREFERENCES.

The placement of the asterisk (`*`) has no bearing on how the code is interpreted; but the warning reflects the fact that the compiler believes we might be confused as to what we intended vs what it's doing. In this case, it's correct. That's why `clang-format` rejiggered all of the pointer declarations to place the asterisk (`*`) with the _type_, and not the _variable_.

Here is what initializer means, _according to the compiler_:

```c
char* p = NULL; // '\0' == 0x00 == NULL
```

This is _very_ different than initializing to a one-character literal with '\0' as that character.

I can fix this by being intentional about what we mean, but in this case, it's unnecessary. It's first usage, as we saw, is an assignment to `code`. It never gets a chance to test for, or dereference, the NULL value initially stored in `p`. So I can dispense with the initial declaration and declare it at the moment we assign `code` to it.

```c
char* p = code;
while (*p != '\0') {
    *string = *p++;
    *++string = '\0';
}
```

There will be howling and gnashing of teeth from old-school C devs, who placed all stack declarations at the top of the function _because they had to_. This hasn't been the case in decades, now (as of C99). Accusations that this is a "C++-ism" are misguided, as this syntax change was introduced in the same C Standard that explicitly broke C++ compatibility. I won't wade into the arguments and counter-arguments; I simply state that, going forward, I will adhere to my personal preference that _small_ and _limited use_ variables are declared in context. I want to see a variable's declaration and use on the same screen.

> Exception: mega-buffers declared on the stack with `MAX_STRING_LENGTH` need to stay put. I don't even like that they exist; I want them to remain a perpetual eyesore until MSVC migration forces me to do something out them.

> Further note: GCC and Clang both have `-std=c99` enabled by default, which allows this behavior. MSVC, as we will see, does _not_. We will need to set a compiler flag to enable C99, which it implements in a inconsistent and incomplete way.

And with that minor change, ROM now compiles under Clang without warnings. Now it's time to move from Linux to Windows.

## Compiling for Cygwin on Windows

The first natural build platform on Windows is [Cygwin](https://www.cygwin.com/) (which has basically the same website it had when I first started using it in 1999; at some point it seems they added rounded borders). 

What is Cygwin? I'll let it speak for itself:

> Cygwin is:
> - a large collection of GNU and Open Source tools which provide functionality similar to a Linux distribution on Windows.
> - a DLL (cygwin1.dll) which provides substantial POSIX API functionality.
> 
> Cygwin is not:
> - a way to run native Linux apps on Windows. You must rebuild your application from source if you want it to run on Windows.
> - a way to magically make native Windows apps aware of UNIX functionality like signals, ptys, etc. Again, you need to build your apps from source if you want to take advantage of Cygwin functionality.

It's not an emulator, it's an API shim for GNU software. So, while I will using the same compiler (GCC) that I used on Linux, it will still be _Windows_, and that means there will be differences. Some of these differences will manifest in header files that Cygwin needs its own bespoke implementation.

### Required Packages

To compile ROM under Cygwin, these packages are required:
- `gcc-core`
- `make`
- `libcript-devel`

If you select them during installation, all the necessary dependencies will be selected, as well.

### Building

Because I am using GCC, I can use the default `Makefile` during build like so:

```
make
```

Interesting. There are far, far more warnings transitioning between platforms than transitioning between compilers on the same platform. These fall however, into very large buckets of similar warnings. The solution for almost all of them is the same.

### Spaces, Locales, and `char`s, Oh My!

```
$ gcc -c -Wall -O -g  act_info.c
In file included from act_info.c:40:
act_info.c: In function ‘do_description’:
act_info.c:2213:28: warning: array subscript has type ‘char’ [-Wchar-subscripts]
 2213 |             while (isspace(*argument))
      |                            ^~~~~~~~~
```

This is new. Given that this GCC 11.3 (the same version I used in Linux), why am I getting a new error? And where is this so-called "array subscript" it's complaining about? Like all character-type testing functions in `ctype.h`, `isspace()` takes an `int`, not a `char`. Why? Because what values constitute "spaces" depends on the system's "locale" (`LC_TYPE`), and some locales are multi-byte.

However, that doesn't explain why this was kosher on Linux, and not in Cygwin (but, as I said, same compiler). First, let's look at how the `ctype.h` include in Cygwin defines `isspace()`:

```c
/* These macros are intentionally written in a manner that will trigger
   a gcc -Wall warning if the user mistakenly passes a 'char' instead
   of an int containing an 'unsigned char'.  Note that the sizeof will
   always be 1, which is what we want for mapping EOF to __CTYPE_PTR[0];
   the use of a raw index inside the sizeof triggers the gcc warning if
   __c was of type char, and sizeof masks side effects of the extra __c.
   Meanwhile, the real index to __CTYPE_PTR+1 must be cast to int,
   since isalpha(0x100000001LL) must equal isalpha(1), rather than being
   an out-of-bounds reference on a 64-bit machine.  */
#define __ctype_lookup(__c) ((__CTYPE_PTR+sizeof(""[__c]))[(int)(__c)])

#define	isalpha(__c)	(__ctype_lookup(__c)&(_U|_L))
#define	isupper(__c)	((__ctype_lookup(__c)&(_U|_L))==_U)
#define	islower(__c)	((__ctype_lookup(__c)&(_U|_L))==_L)
#define	isdigit(__c)	(__ctype_lookup(__c)&_N)
#define	isxdigit(__c)	(__ctype_lookup(__c)&(_X|_N))
#define	isspace(__c)	(__ctype_lookup(__c)&_S)
#define ispunct(__c)	(__ctype_lookup(__c)&_P)
#define isalnum(__c)	(__ctype_lookup(__c)&(_U|_L|_N))
#define isprint(__c)	(__ctype_lookup(__c)&(_P|_U|_L|_N|_B))
#define	isgraph(__c)	(__ctype_lookup(__c)&(_P|_U|_L|_N))
#define iscntrl(__c)	(__ctype_lookup(__c)&_C)
```

The comment tells us everything we need to know: `char`, being signed, is incorrect (however, I'm going to argue that this is something of a hack, as provoking `-Wchar-subscripts` yields an obtuse and unhelpful warning). These functions intend to force us to use locale-aware data types.

_However_... I don't care. Because ROM, like all other MUDs, is a glorified telnet server, it strictly uses 7-bit ASCII (also referred to conventionally as "C Locale" or "POSIX Locale") for data, and reserves the sign bit for special control characters. For that reason, I have no intention of adding support for any local that requires 8 bits or multibyte characters. So I will do what I must to keep my default string type as `char*`, and be happy with it.

Now, why was my Linux set up okay with this? Well, as it turns out, the `ctype.h` header is very different:

```c
# define isspace(c)	__isctype((c), _ISspace)
```

```c
# define __isctype(c, type) \
  ((*__ctype_b_loc ())[(int) (c)] & (unsigned short int) type)
```

```c
/* These are defined in ctype-info.c.
   The declarations here must match those in localeinfo.h.

   In the thread-specific locale model (see `uselocale' in <locale.h>)
   we cannot use global variables for these as was done in the past.
   Instead, the following accessor functions return the address of
   each variable, which is local to the current thread if multithreaded.

   These point into arrays of 384, so they can be indexed by any `unsigned
   char' value [0,255]; by EOF (-1); or by any `signed char' value
   [-128,-1).  ISO C requires that the ctype functions work for `unsigned
   char' values and for EOF; we also support negative `signed char' values
   for broken old programs.  The case conversion arrays are of `int's
   rather than `unsigned char's because tolower (EOF) must be EOF, which
   doesn't fit into an `unsigned char'.  But today more important is that
   the arrays are also used for multi-byte character sets.  */
extern const unsigned short int **__ctype_b_loc (void)
     __THROW __attribute__ ((__const__));
extern const __int32_t **__ctype_tolower_loc (void)
     __THROW __attribute__ ((__const__));
extern const __int32_t **__ctype_toupper_loc (void)
     __THROW __attribute__ ((__const__));
```

"Broken old programs" kinda hurts, but it's nice to know GNU is still thinking about me. But now I'm doubting myself. Should I keep ROM's strings as `char`, or move to `unsigned char`? In the end, it's a judgement call. But I know this: making such a wide-reaching change will make the code more "correct" to absolutely no benefit, whatsoever; we simply cannot use 8-bit or multi-byte on the wire. 

Therefore, I will take the easy way out by adding a new file called `strings.h`:

```c
///////////////////////////////////////////////////////////////////////////////
// strings.h
// 
// Helper functions for string handling.
///////////////////////////////////////////////////////////////////////////////

#pragma once
#ifndef ROM__STRINGS_H
#define ROM__STRINGS_H

// All strings in ROM are 7-bit POSIX Locale, with the sign bit reserved for 
// special telnet characters. These macros keep 'ctype.h' functions copacetic
// with char* strings under -Wall.

#ifdef __CYGWIN__
    #define ISSPACE(c) isspace((unsigned char)c)
#else
    #define ISSPACE(c) isspace(c)
#endif

#endif // !ROM__STRINGS_H
```

> I made this a separate file because, as I said, I fundamentally disagree with the existence of `merc.h` as a catch-all mega-header. I'm making a small, focused header that will eventually grow as I add new string-related functionality. I will probably create more little headers like this as time goes on.

I then include it in `act_info.c`:

```c
#include "strings.h"
```

And then replace the call like so:

```c
 while (ISSPACE(*argument))
```

Is it an ideal solution? No. But it works within acceptable parameters.

Note that this only takes care of `isspace()` for this one call. I will leave replacing other instances of `isspace()`, as well as the other character testing functions in `ctype.h`, as an exercise to the reader.

### Missing `crypt()`... Again

The next one kind of stinks to have to deal with, again:

```
act_info.c: In function ‘do_password’:
act_info.c:2434:16: warning: implicit declaration of function ‘crypt’ [-Wimplicit-function-declaration]
 2434 |     if (strcmp(crypt(arg1, ch->pcdata->pwd), ch->pcdata->pwd)) {
      |                ^~~~~
act_info.c:2434:16: warning: passing argument 1 of ‘strcmp’ makes pointer from integer without a cast [-Wint-conversion]
 2434 |     if (strcmp(crypt(arg1, ch->pcdata->pwd), ch->pcdata->pwd)) {
      |                ^~~~~~~~~~~~~~~~~~~~~~~~~~~~
      |                |
      |                int
```

When I see `-Wimplicit-function-declaration` followed immediately by `-Wint-conversion`, I know that I am missing a library reference. In this case, `libcrypt`. Implicit declarations always assume `int` as the data type of all unknown variables, including function return types and parameters. So, where's `libcrypt`? I made sure to install `libcrypt-devel` to take care of this, right?

This is another area where Cygwin reminds us that it is not UNIX. Notice before I said that this is a missing library reference, _not_ a missing library, itself. This isn't a linker error; it's the compiler not having a definition for `crypt()`, at all. In UNIX, `crypt()` is defined in `unistd.h`, but I need something else in `act_info.c` for Cygwin:

```c
#ifdef __CYGWIN__
#include <crypt.h>
#endif
```

I don't just throw these platform-specific `#include`s all willy-nilly; I make sure that it doesn't pollute the other build platforms. This is a convention the authors of ROM followed; just take a look at `comm.c`:

```c
#if defined(apollo)
...
#endif

#if defined(unix)
...
#endif

#if defined(macintosh)
...
#endif

#if defined(_AIX)
...
#endif

#if defined(__hpux)
...
#endif

#if defined(interactive)
...
#endif

#if defined(MIPS_OS)
...
#endif

#if defined(MSDOS)
...
#endif

#if defined(NeXT)
...
#endif

#if defined(sequent)
...
#endif

#if defined(sun)
...
#endif

#if defined(ultrix)
...
#endif
```

The nice thing is, almost all of those are now irrelevant. And eventually we will get around to cleaning them out. Not yet, though!

### Drumroll, Please...

Those two issues largely cleared up all the warnings I had. Let's see how it builds:

```
$ make clean && make
rm -f rom
rm -f *.o
gcc -c -Wall -O -g  act_comm.c
gcc -c -Wall -O -g  act_enter.c
gcc -c -Wall -O -g  act_info.c
gcc -c -Wall -O -g  act_move.c
gcc -c -Wall -O -g  act_obj.c
gcc -c -Wall -O -g  act_wiz.c
gcc -c -Wall -O -g  alias.c
gcc -c -Wall -O -g  ban.c
gcc -c -Wall -O -g  comm.c
gcc -c -Wall -O -g  const.c
gcc -c -Wall -O -g  db.c
gcc -c -Wall -O -g  db2.c
gcc -c -Wall -O -g  effects.c
gcc -c -Wall -O -g  fight.c
gcc -c -Wall -O -g  flags.c
gcc -c -Wall -O -g  handler.c
gcc -c -Wall -O -g  healer.c
gcc -c -Wall -O -g  interp.c
gcc -c -Wall -O -g  note.c
gcc -c -Wall -O -g  lookup.c
gcc -c -Wall -O -g  magic.c
gcc -c -Wall -O -g  magic2.c
gcc -c -Wall -O -g  music.c
gcc -c -Wall -O -g  recycle.c
gcc -c -Wall -O -g  save.c
gcc -c -Wall -O -g  scan.c
gcc -c -Wall -O -g  skills.c
gcc -c -Wall -O -g  special.c
gcc -c -Wall -O -g  tables.c
gcc -c -Wall -O -g  update.c
rm -f rom
gcc -O -g -o rom act_comm.o act_enter.o act_info.o act_move.o act_obj.o act_wiz.o alias.o ban.o comm.o const.o db.o db2.o effects.o fight.o flags.o handler.o healer.o interp.o note.o lookup.o magic.o magic2.o music.o recycle.o save.o scan.o skills.o special.o tables.o update.o -lcrypt
```

Looking good, so far! Let's give it a whirl:

```
cd ../area
../src/rom.exe 4000
```

And my boot log:

```
Tue May  9 22:48:33 2023 :: [*****] BUG: Fix_exits: 10525:1 -> 10535:3 -> 10534.
Tue May  9 22:48:33 2023 :: [*****] BUG: Fix_exits: 3458:2 -> 3472:0 -> 10401.
Tue May  9 22:48:33 2023 :: [*****] BUG: Fix_exits: 8705:4 -> 8706:5 -> 8708.
Tue May  9 22:48:33 2023 :: [*****] BUG: Fix_exits: 8717:2 -> 8719:0 -> 8718.
Err: obj an icicle (9227) -- 28, mob the Ice Bandit (9228) -- 24
Err: obj elemental wand of wind and air (9218) -- 27, mob an alchemist (9234) -- 13
Err: obj an ice staff (9216) -- 25, mob a baby rainbow dragon (9235) -- 16
Err: obj an ice staff (9216) -- 25, mob a puddle (9214) -- 8
Err: obj elemental wand of fire (9215) -- 16, mob a flame (9215) -- 4
Err: obj elemental wand of fire (9215) -- 16, mob a flame (9215) -- 4
Err: obj an elemental rod of earthquake (9217) -- 7, mob a small rock (9217) -- 3
Err: obj elemental wand of wind and air (9218) -- 27, mob a small spark (9218) -- 4
Err: obj elemental wand of wind and air (9218) -- 27, mob an eddie (9225) -- 2
Err: obj a wet noodle (8010) -- 5, mob a Futsie (8002) -- 17
Tue May  9 22:48:33 2023 :: ROM is ready to rock on port 4000.
```

Seems to be working fine. Windows gave me a prompt to open a firewall port, which I accepted. Let's see how it looks when we connect:

```
THIS IS A MUD BASED ON.....

                                ROM Version 2.4 beta

               Original DikuMUD by Hans Staerfeldt, Katja Nyboe,
               Tom Madsen, Michael Seifert, and Sebastian Hammer
               Based on MERC 2.1 code by Hatchet, Furey, and Kahn
               ROM 2.4 copyright (c) 1993-1998 Russ Taylor

By what name do you wish to be known? 
```

Awesome. We now have ROM running, warning-free, on three compilers on two different platforms! But the next one is going to be the biggest bugbear: MSVC on Windows. But it's an exercise worth doing.

([Here is the code](https://github.com/bfelger/Mud98/tree/050e23e08a81f86ec8f7352b6f18c8ba55eb8fdb) with updates from this post.)

Next: [Part 3](pt-3-dusting-off-cobwebs)

Copyright 2023, Brandon Felger
