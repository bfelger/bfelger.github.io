---
layout: default
title:  "Resurrecting ROM Pt. 3 - Dusting Off the Cobwebs"
date:   2023-05-10 20:00:00 -0500
categories: mud C Unix
---
# Resurrecting ROM Pt. 3 &mdash; Dusting Off the Cobwebs

_Skip this if you still need to build ROM on an HP 9000/300 from 1988._

This post picks up where [Part 2](pt-2-compile-clang) left off. [Here is the code](https://github.com/bfelger/rom/tree/050e23e08a81f86ec8f7352b6f18c8ba55eb8fdb) that resulted from that work. So far, I have ROM building in GCC on Ubuntu without errors or warnings.

I know I said Part 3 was going to be porting to MSVC, but upon reflection, I believe significant work needs to be done to make that work as painless as possible. 

I will end up splicing in many, many macros to separate POSIX and Win32. Right now, however, there are still hundreds of macros for other, older platforms, many of which are no longer in use. Getting to a "clean" slate with as few legacy architecture macros will make my task much easier.

However, I will lose the ability to build ROM on the Apple Macintosh LC II (circa 1992). I will do my best not to lose sleep over it.

## Trimming the Fat

Getting rid of the legacy preprocessor macros will be a slightly laborious task. To make it easier, I will just be doing "Search in Files" for the legacy platforms and removing them one at a time. I will also remove guards from code that is currently our "one true path" (like `unix`). Here are a few examples of what I will be doing, here (from `comm.c`):

### Old Architectures

```c
#if defined(macintosh)
#include <types.h>
#else
#include <sys/time.h>
#include <sys/types.h>
#endif
```

We no longer need Macintosh support, so the entire block will be reduced to the `#else` block:

```c
#include <sys/time.h>
#include <sys/types.h>
```

Because these are no longer guarded, they can be moved into the other `#include`'s, which are sorted in alphabetical order,

> The ordering of `#include`s is a holy war I'm not interested in waging. But the existing code does it that way, and it appeals enough to my OCD that I am unmotivated to change it.

Meanwhile, legacy platform `#ifdef`s with no attendant `#else`s can be deleted entirely:

```c
#if defined(sun)
#undef MALLOC_DEBUG
#endif
```

Look for surrounding comments and code that are mooted by the removal of these legacy macros:

```c
/*
 * Signal handling.
 * Apollo has a problem with __attribute(atomic) in signal.h,
 *   I dance around it.
 */
#if defined(apollo)
#define __attribute(x)
#endif
```

These comments can be removed, also.

### Watch for `#ifndef` and `!defined()`

This snippet is in `note.c`:

```c
#if !defined(macintosh)
extern int _filbuf args((FILE*));
#endif
```

When chopping code, it's easy to allow a crazed bloodlust to take over and miss a NOT condition. In this case, I want to unguard the `extern` statement. 

### Establish Unix as the New Baseline

Because I want the code to assume UNIX for the time being (that will change; right now we just want the code to be clean), I can remove the guards on `unix` blocks:

```c
#if defined(unix)
#include <signal.h>
#endif
```

In this case, I add...

```c
#include <signal.h>
```

...to the other `#include`s. The next part of this series, where I add MSVC support, this particular header will be moving back out; but I want it to be in the right spot when I do so.

Some legacy macros are boolean OR'd with my default `unix`:

```c
#if defined(macintosh) || defined(unix)
    if (strlen(name) > 12) return FALSE;
#endif
```

I need to be sure I keep the "good" code:

```c
if (strlen(name) > 12) 
    return FALSE;
```

### Wait, is that Linux?

In `comm.c`, we have this:

```c
#if !defined(OLD_RAND)
#if !defined(linux)
long random();
#endif
void srandom(unsigned int);
int getpid();
time_t time(time_t* tloc);
#endif
```

Remember how I said ROM's progenitors pre-dated Linux? Well, ROM's last release (1998) was well after the establishment of Linux, so this should come as no surprise. Now, what code should I keep from this?

Answer: None of it. These definitions should come from a system header file.

In the words of Sally from Mystery Men, _"JUNK IT!"_ I'll fix the errors that arise, later.

### `crypt()` Rears Its Ugly Head

In `merc.h`, my culling brought me to this beauty:

```c
/*
 * OS-dependent declarations.
 * These are all very standard library functions,
 *   but some systems have incomplete or non-ansi header files.
 */
#if defined(_AIX)
char* crypt args((const char* key, const char* salt));
#endif

#if defined(hpux)
char* crypt args((const char* key, const char* salt));
#endif

#if defined(linux)
char* crypt args((const char* key, const char* salt));
#endif

#if defined(macintosh)
#define NOCRYPT
#if defined(unix)
#undef unix
#endif
#endif
```

It continues from there. But essentially, ROM was explicitly defining `crypt()` instead of bringing in the relevant header. I _caught_ this in Cygwin, because that build platform doesn't match any of the legacy macros, and therefore didn't explicitly define `crypt()`.

I will delete this entire block of `merc.h`. This will probably cause compiler errors, but I'll get to that when I'm done culling.

### More Library Calls

Plumbing through `merc.h`, I found more standard library/Unix calls:

```c
/* system calls */
int unlink();
int system();
```

They gotta go. I want to pull in the correct declaration from the relevant headers.

Likewise, this:

```c
/*
 * Short scalar types.
 * Diavolo reports AIX compiler has bugs with short types.
 */
#if !defined(FALSE)
#define FALSE 0
#endif

#if !defined(TRUE)
#define TRUE 1
#endif

#if defined(_AIX)
#if !defined(const)
#define const
#endif
typedef int sh_int;
typedef int bool;
#define unix
#else
typedef short int sh_int;
typedef unsigned char bool;
#endif
```

Some of the compilers Merc/Diku/ROM was written against were quite lacking in basic types. But all of this has been standardized for other 30 years, and it is no longer necessary to have any of this.

### On `/dev/null`

In `merc.h`, removing `MSDOS` and unguarding `unix` gives me this in the main code:

```c
#define NULL_FILE       "/dev/null"     // To reserve one stream
```

There is a lot of code that was in the `unix` path that is actually now a part of the Win32 API and/or platform, but this isn't one of them. I'm puttig a TODO here to make sure I touch on this when I port to MSVC:

```c
// TODO: Research how to reserve a stream in MSVC without /dev/null.
#define NULL_FILE       "/dev/null"     // To reserve one stream
```

### Summary

So, in short:

Platform macros to un-guard:
- `unix` &mdash; The baseline. Where we need to deviate, we will guard with that specific platform (like `__CYGWIN__` or whatever we use for MSVC)

Platform macros to keep:
- `__CYGWIN__`  &mdash; Considering we just added it in the previous post in this series.

Platform macros to cull:
- `_AIX`
- `apollo`
- `__hpux`
- `interactive` &mdash; It has one guard, and no usages.
- `linux` &mdash; It was only used in one place, where it was used to explicitly declare library functions. I axed it.
- `macintosh` &mdash; Mac and DOS were supported only in single-user "console" mode. I don't think it's helpful to try to keep this functionality, and both of these platforms are now dead.
- `MIPS_OS`
- `MSDOS` &mdash; See `macintosh` above.
- `NeXT`
- `sequent`
- `sun`
- `SYSV` &mdash; This one is sometimes OR'd with telnet options I will otherwise keep. In particular, I can assume `!defined(SYSV)` is always `true`.
- `ultrix`

## Miscellaneous Cleanup

There are a few more spring-cleaning items I want to take care of that will make the port to MSVC easier.

### Names of Things

Now that we are using the old `unix` path as the default, One True Path(tm), there is one thing I'd like to change. The `unix`, `MSDOS` and `macintosh` macros toggled between two different implementations of the main game loop: `game_loop_msdos()` (for `MSDOS` and `macintosh`) and `game_loop_unix()` (for everything else).

`game_loop_msdos()` is a serverless, single-player console game loop. This is because those platforms _had no built-in network stack_. 

> So it's not a MUD; it's a SUD. _(har har)_

But all of our target platforms can run in server mode, and will run soon-to-be misnamed `game_loop_unix()`. The solution is to simply rename it to `game_loop()` everywhere it appears.

### A Couple Random Things...

I mean, _`random()`_ things _(har har)_

This is in `db.c`:

```c
/*
 * I've gotten too many bad reports on OS-supplied random number generators.
 * This is the Mitchell-Moore algorithm from Knuth Volume II.
 * Best to leave the constants alone unless you've read Knuth.
 * -- Furey
 */

/* I noticed streaking with this random number generator, so I switched
   back to the system srandom call.  If this doesn't work for you,
   define OLD_RAND to use the old system -- Alander */

#if defined(OLD_RAND)
static int rgiState[2 + 55];
#endif
```

There is a lot of code gated by `OLD_RAND`, and it needs to go. Most platforms now put quite a bit of work into solid implementations of PRNG's. And I may even visit this topic myself, eventually.

And while I'm at it, I'm getting rid of any trace of the `TRADITIONAL` flag.

## How Did We Do?

I'm still on my Cygwin environment, but I'll go ahead and give it a try:

```
make clean && make
```

Holy Cannoli, that's a lot of errors.

### Missing `bool`

Here's the first one:

```
gcc -c -Wall -O -g  act_comm.c
In file included from interp.h:32,
                 from act_comm.c:29:
merc.h:73:9: error: unknown type name ‘bool’
   73 | typedef bool SPEC_FUN args((CHAR_DATA * ch));
      |         ^~~~
```

No `bool`? And if I had to guess, `true` and `false` are missing, too.

All of these were introduced in C99 because of how commonly they were implemented in user code (as ROM itself does [or did, until I removed it]). However, most compilers have back-ported this addition to previous standards.

In C99, the actual, built-in type is `_Bool`, but the standard also requires `bool`, `true`, and `false` to be defined in `stdbool.h`, which I will add to `merc.h`:

```C
#include <stdbool.h>
```

I placed it in `merc.h` (despite my repeated diatribes against it) because it is the highest-level header in the every code file, and itself uses `bool`.

### Handling Modern Integers

The next error is fairly pervasive:

```
In file included from interp.h:32,
                 from act_comm.c:29:
merc.h:258:5: error: unknown type name ‘sh_int’
  258 |     sh_int ban_flags;
      |     ^~~~~~
```

`sh_int` was defined in `merc.h` as a `short int` on any non-AIX system (or `int` on AIX). The _important_ thing is it be a 16-bit signed integer. Thankfully, we have a type for that, now: `int16_t`. It's one of many types added by C99 that I want to avail myself of:

| C99 Type      | 64-bit Native Type   | Legacy 16-bit Type   |
| :---          | :---                 | :---                 |
| `int8_t`      | `char`               | `char`               |
| `uint8_t`     | `unsigned char`      | `unsigned char`      |
| `int16_t`     | `short`              | `short`/`int`        |
| `uint16_t`    | `unsigned short`     | `unsigned short`/`int`|
| `int32_t`     | `int`/`long`        | `long`                |
| `uint32_t`    | `unsigned int`/`long`| `unsigned long`      |
| `int64_t`     | `long long`          |  |
| `uint64_t`    | `unsigned long long` |  |

So why the change? Well, see, the native types listed here are only correct for modern 32-bit x86 and AMD64. Those values are very fluid, depending on architecture.

For instance, on a 16-bit system, and `int` is a `int16_t`, and a `long` is a `int32_t`. On those systems, `short` and `int` are the same (`int16_t`). Likewise, on 32-bit systems, `int` and `long` are the same (`int32_t`).'

And why are `int` and `long` so varying by platform? It _usually_ has to do with what the architecture considers the most optimally-sized data type for the CPU's microcode to deal with. In C, the intention is that, whatever your platform, picking `int` for a transient integer is almost always the right call.

But when we need to store things on disk, and send them over the wire, we need static, predictable sizes. So I'm going to end up with a smattering of native types and C99 types, and that is _perfectly fine._

But what to do about `sh_int`? No matter what the architecture is, a `short int` is an `int16_t`. This is the correct type to use in place of `sh_int`.

First, I will add `stdint.h` to `merc.h`:

```C
#include <stdint.h>
```

Now, I _could_ do a `#define` or `typedef`, but I don't want to deal with `sh_int`, as that is not what I'm used to, and it's not the standard format. So instead I will do a global find/replace to swap out `sh_int` for `int16_t` (all 272 instances of it).

> `stdint.h` defines more than just these; there are also "fast" integers and integer pointers. I'll evaluate each of these when the time comes.

And with that, the vast majority of errors are gone.

### More Missing Library References

This one I probably did, myself, while refactoring `#include`s:

```c
In file included from interp.h:32,
                 from act_comm.c:29:
merc.h:504:5: error: unknown type name ‘time_t’
  504 |     time_t date_stamp;
      |     ^~~~~~
```

Adding this to `merc.h` fixes it:

```c
#include <time.h>
```

### Replacing the Old `BOOL`s

The next two errors reveal that we have some of the old, non-`stdbool.h` macros hanging around:

```c
act_comm.c: In function ‘do_delete’:
act_comm.c:57:42: error: ‘FALSE’ undeclared (first use in this function)
   57 |             ch->pcdata->confirm_delete = FALSE;
      |                                          ^~~~~
act_comm.c:57:42: note: each undeclared identifier is reported only once for each function it appears in
act_comm.c:63:31: error: ‘TRUE’ undeclared (first use in this function)
   63 |             stop_fighting(ch, TRUE);
      |                               ^~~~
```

The fix is simple enough: a global replacement of `TRUE` and `FALSE` with `true` and `false`, respectively, will eliminate these errors (I already included `stdbool.h` in `merc.h`).

### Missing POSIX Declarations

I said was was going to break it, and I did:

```
act_comm.c: In function ‘do_delete’:
act_comm.c:65:13: warning: implicit declaration of function ‘unlink’ [-Wimplicit-function-declaration]
   65 |             unlink(strsave);
      |             ^~~~~~
db.c: In function ‘init_mm’:
db.c:2717:26: warning: implicit declaration of function ‘getpid’ [-Wimplicit-function-declaration]
 2717 |     srandom(time(NULL) ^ getpid());
      |                          ^~~~~~
save.c: In function ‘load_char_obj’:
save.c:642:9: warning: implicit declaration of function ‘system’ [-Wimplicit-function-declaration]
  642 |         system(buf);
      |         ^~~~~~
```

Thanks to `man` pages, these are easy fixes.

I add this to `act_comm.c`, `ban.c`, and `db.c`:

```c
#include <unistd.h>
```

And I add this to `save.c`:

```c
#include <stdlib.h>
```

### Unused Result

Everything seems fine on the Cygwin side. But after compiling under GCC on Ubuntu, I have this warning, from `load_char_obj()` in `save.c`:

```c
save.c:643:9: warning: ignoring return value of ‘system’ declared with attribute ‘warn_unused_result’ [-Wunused-result]
  643 |         system(buf);
      |         ^~~~~~~~~~~
```

This error comes from the code that zips up game files:

```c
sprintf(buf, "gzip -dfq %s", strsave);
if (!system(buf)) {
```

`system` returns an `int`, and that value has useful information: it's `-1` if the system command fails. This was a helpful warning, and I fix it by testing the result like so:

```c
sprintf(buf, "gzip -dfq %s", strsave);
if (!system(buf)) {
    sprintf(buf, "ERROR: Failed to zip %s (Error Code: %d).", strsave, errno);
    log_string(buf);
}
```

But I also have to add `errno.h` to `save.c`:

```c
#include <errno.h>
```

I also have to expand the buffer to accommodate the log string:

```c
char buf[MAX_INPUT_LENGTH+50]; // To handle strsave + format
```

## Coda

And with that, I have exorcised all the demons of legacy platforms (_that I know of_) and strengthened ROM's reliance on system and library headers instead of ad hoc (and outdated) definitions.

There is yet more work to be done to set myself up for success in porting to MSVC, and that is getting rid of Make and adopting CMake. That will be Part 4 of the "Resurrecting ROM" series.

([Here is the code](https://github.com/bfelger/rom/tree/addcc5c67e81940dff838f45e12fbae22c75ecff) with updates from this post.)

Copyright 2023, Brandon Felger