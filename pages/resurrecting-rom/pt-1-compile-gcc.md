---
layout: default
title:  "Resurrecting ROM Pt. 1 - Getting it to Compile Under Linux With GCC"
date:   2023-05-06 20:30:00 -0500
categories: mud C gcc
---
# Resurrecting ROM Pt. 1 &mdash; Getting it to Compile Under Linux With GCC

_A short exercise in code necromancy (2023-05-06)_

## Graveyard Digging

I've collected the archaeological relics of ROM 2.4b6 and Lope's ColoUr code [here](assets/Rom24andColour.zip). It's a ZIP file, but inside are two "tar-balls" that you can open with `tar -xzf` on a Linux box or Cygwin (or third-party Windows unzipper of your choice).

If the idea of manually fixing dozens of patch merge conflicts isn't your cup of tea, you can follow along by grabbing [my copy](https://github.com/bfelger/rom/tree/ba26f981060201c3ef4db25a91d0ffd65d680a6c) with the patches already applied.

## Compiling with GCC

ROM was developed and tested using GCC on a variety of weird UNIX machines (its progenitors, Merc and Diku, predated Linux by several years), so Linux gives me my best shot at compiling it out of the box.

So, that's what I will try (using GCC 11.3 on Ubuntu 22.04).\:

```
make
```
Yikes! That's a lot of errors. I have to take it one file at a time, and one error at a time.

I'll start with the first file listed in order of compilation, `act_info.c`:

```
gcc -c -Wall -O -g  act_comm.c
```

And here we go! Now I begin the laborious task of fixing defects. Let's take them each in turn.

### Head(er) Aches

Here's the first error, with wide-reaching implications:

```
In file included from act_comm.c:34:
interp.h:31:18: error: expected ‘=’, ‘,’, ‘;’, ‘asm’ or ‘__attribute__’ before ‘args’
   31 | void do_function args((CHAR_DATA * ch, DO_FUN* do_fun, char* argument));
      |                  ^~~~
```

Here is the offending code:

```c
/* wrapper function for safe command execution */
void do_function args((CHAR_DATA * ch, DO_FUN* do_fun, char* argument));
```

Here's the source of the problem in `merc.h`:

```c
/*
 * Accommodate old non-Ansi compilers.
 */
#if defined(TRADITIONAL)
#define const
#define args(list)             ()
#define DECLARE_DO_FUN(fun)    void fun()
#define DECLARE_SPEC_FUN(fun)  bool fun()
#define DECLARE_SPELL_FUN(fun) void fun()
#else
#define args(list)             list
#define DECLARE_DO_FUN(fun)    DO_FUN fun
#define DECLARE_SPEC_FUN(fun)  SPEC_FUN fun
#define DECLARE_SPELL_FUN(fun) SPELL_FUN fun
#endif
```

Because ROM's predecessor, Merc, was being compiled on dozens of archaic systems, it had to deal with old compilers. The earliest C compilers didn't prototype the entire function signature; just the identifier. Therefore, the `TRADITIONAL` flag lets you replace function arguments with a bare `()`.

Now, I don't have `TRADITIONAL` defined (because it's not 1985 anymore), so `args(list)` becomes just `list`. The very first standard of C ("ANSI" C) requires full function signatures in prototypes, however; hence this preproc macro. So I _should_ be good, right?

I notice that `interp.h` doesn't include `merc.h`, which means that `act_info.c` is including both, here:

```c
#include "interp.h"
#include "lookup.h"
#include "magic.h"
#include "merc.h"
#include "recycle.h"
#include "tables.h"
```
Can you spot what's wrong?

The files are out of order. `merc.h` needs to be above `interp.h`. Or, better yet; it need to be referenced directly _from_ `interp.h`. But it isn't, and why?

Take a look at the header files, and it may become apparent. There is something missing from each one. It's the "include guard". Because of this omission, the authors take great pains to include each file only once. I can dispense with that, now.

First, I add this to `interp.h`:

```c
#include "merc.h"
```

Now, to avoid any issues, I add this to `merc.h`:

```c
#pragma once
#ifndef ROM__MERC_H
#define ROM__MERC_H
```

Hey now, why are there two include guards? Both should be recognizeable; the traditional, portable `#define` and Microsoft's propriatary, non-portable `#pragma once`.

This is called a "double guard", and there is a very good reason for doing it this way. `#pragma once` is now implemented in most big-name C compilers, and for those that do _not_ implement it, it's a NOOP. This pragma makes it so that the compiler remembers opening _this exact file_, and never opens it again. Problem: the compiler can still include the file _if_ it comes from a different path.

The traditional include guard is more robust; multiple copies of the file will use the same `#define`, and thus only the first will be opened. Problem: the include will be opened every time it is requested, only to bail once it hits the guard.

My approach is the best of both worlds. The same file at that path is never opened again, and other copies of that same header are opened (and bailed from) exactly once.

I need to make sure to close the guard at the end of the file:

```c
#endif // !ROM__MERC_H
```

It's a useful bit of boilerplate, so I will go ahead and add it to all of the header files, also adding the essential `merc.h` _inside_ the guard, like so:

```c
#pragma once
#ifndef ROM__INTERP_H
#define ROM__INTERP_H

#include "merc.h"
```

That nukes a whole class of errors for me.

### Standard Headers

Here's the next error to deal with:

```
In file included from interp.h:32,
                 from act_comm.c:34:
merc.h:1939:8: error: unknown type name ‘FILE’
 1939 | extern FILE* fpReserve;
      |        ^~~~
```

`merc.h` has no reference to `stdio.h`, so it has no idea what `FILE` is. Presumedly, the code files that actually _call_ these functions will include `stdio.h`, and perhaps in the days of yore, the code files that _didn't_ would elide over these incomplete definitions. That is definitely not the case, today.

Note that this is not the only place in `merc.h` that references `FILE`:

```c
char fread_letter args((FILE * fp));
int fread_number args((FILE * fp));
long fread_flag args((FILE * fp));
char* fread_string args((FILE * fp));
char* fread_string_eol args((FILE * fp));
void fread_to_eol args((FILE * fp));
char* fread_word args((FILE * fp));
```

This is a whole class of functions of related purpose. I fundamentally disagree with these function signatures being propagated to every translation unit. I want to separate these out into a dedicated header, but right now I just want it to build. Therefore, I'll hack it for now by adding this to `merc.h`:

```c
#include <stdio.h>
```

> Remember that everything goes _inside_ the include guards.

I don't like propagating standard libraries to every translation unit. But the work to fix that can be deferred until we have everything building. Now that we've gotten that licked, we move on.

### Struct Definitions

Here's the next error:

```
In file included from act_comm.c:37:
tables.h:35:31: error: array type has incomplete element type ‘struct clan_type’
   35 | extern const struct clan_type clan_table[MAX_CLAN];
      |                               ^~~~~~~~~~
```

Looking in `tables.h`, I see a list of `extern`s using types defined immediately after. Again, this is a case of older compilers being more than willing to defer type definition. This is allowed, but you must at least provide a _declaration_:

```c
struct clan_type;
```

But this is poor organization, and it needs to be fixed. The solution, therefore, is to make sure that these headers follow a set sequence:

```
┌───────────────────────┐
│ #include Guard        │
├───────────────────────┤
│ Macros                │
├───────────────────────┤
│ struct Definitions    │
├───────────────────────┤
│ Function Declarations │
├───────────────────────┤
│ extern Declarations   │
├───────────────────────┤
│ End #include Guard    │
└───────────────────────┘
```

In `tables.h`, this is trival work; just move the `extern`s to the bottom. Most of the other headers, however, have all these elements in backwards order, and thus are more work.

I'm not touching `merc.h`, however; that header (in my opinion) shouldn't even exist.  I will change it enough to get the code built, and refactor later. In addition to `tables.h`, I also reorder the contents of `db.h`, `interp.h`, and `music.h`.

I'm not touching `telnet.h` with a 10-foot pole. It's fine.

### Eureka!

`act_comm.c` now builds with no errors, and no warnings. What about the rest?

```
make
```

Huh. Well, that list is _much_ more manageable than before. There is only one remaining error (in `comm.c`) and a bunch of warnings. However, I don't want any warnings, either. I'll start with `comm.c` though, and take care of the error, first.

### Standard Libraries II: Electric Boogaloo

There is a pattern in ROM declaring standard library calls explicitly instead of including the relevant header file. That leads to this:

```
comm.c:167:5: error: conflicting types for ‘gettimeofday’; have ‘int(struct timeval *, struct timezone *)’
  167 | int gettimeofday args((struct timeval * tp, struct timezone* tzp));
      |     ^~~~~~~~~~~~
In file included from comm.c:49:
/usr/include/x86_64-linux-gnu/sys/time.h:67:12: note: previous declaration of ‘gettimeofday’ with type ‘int(struct timeval * restrict,  void * restrict)’
   67 | extern int gettimeofday (struct timeval *__restrict __tv,
      |            ^~~~~~~~~~~~
```

Here's the relevant code in `comm.c`:

```c
#if defined(linux)
/*
    Linux shouldn't need these. If you have a problem compiling, try
    uncommenting these functions.
*/
/*
int	accept      args( ( int s, struct sockaddr *addr, int *addrlen ) );
int	bind        args( ( int s, struct sockaddr *name, int namelen ) );
int	getpeername args( ( int s, struct sockaddr *name, int *namelen ) );
int	getsockname args( ( int s, struct sockaddr *name, int *namelen ) );
int	listen      args( ( int s, int backlog ) );
*/

int close args((int fd));
int gettimeofday args((struct timeval * tp, struct timezone* tzp));
int read args((int fd, char* buf, int nbyte));
int select args((int width, fd_set* readfds, fd_set* writefds,
                 fd_set* exceptfds, struct timeval* timeout));
int socket args((int domain, int type, int protocol));
int write args((int fd, char* buf, int nbyte));
#endif
```

Here is a function declaration in `sys/time.c`:

```c
/* Get the current time of day, putting it into *TV.
   If TZ is not null, *TZ must be a struct timezone, and both fields
   will be set to zero.
   Calling this function with a non-null TZ is obsolete;
   use localtime etc. instead.
   This function itself is semi-obsolete;
   most callers should use time or clock_gettime instead. */
#ifndef __USE_TIME_BITS64
extern int gettimeofday (struct timeval *__restrict __tv,
                         void *__restrict __tz) __THROW __nonnull ((1));
#else
# ifdef __REDIRECT_NTH
extern int __REDIRECT_NTH (gettimeofday, (struct timeval *__restrict __tv,
                                          void *__restrict __tz),
                           __gettimeofday64) __nonnull ((1));
# else
#  define gettimeofday __gettimeofday64
# endif
#endif
```

Regardless of whether `__USE_TIME_BITS64` or `__REDIRECT_NTH` is set, the explicit declaration in `comm.c` is wrong. And that's why you don't try to "guess" the function signature in standard libraries; you include the file. As it just so happens, `sys/time.h` _is_ already included, so that function declaration is completely unnecessary.

To get ahead of the potential errors, I'll delete the entire block. This may break more code, but at least it will force me to include the correct standard header files.

Sure enough, we replace one error with another:

```
comm.c: In function ‘main’:
comm.c:375:5: warning: implicit declaration of function ‘close’; did you mean ‘pclose’? [-Wimplicit-function-declaration]
  375 |     close(control);
      |     ^~~~~
      |     pclose
```

On Linux, `close()` lives in `unistd.h` a header file for POSIX (which `close()` is a part of), so we add it to `comm.c`:

```c
#include <unistd.h>
```

### The Fluidity of Signedness

The next error is interesting:

```
comm.c: In function ‘init_descriptor’:
comm.c:763:51: warning: pointer targets in passing argument 3 of ‘getsockname’ differ in signedness [-Wpointer-sign]
  763 |     getsockname(control, (struct sockaddr*)&sock, &size);
      |                                                   ^~~~~
      |                                                   |
      |                                                   int *
In file included from /usr/include/netinet/in.h:23,
                 from /usr/include/netdb.h:27,
                 from comm.c:107:
/usr/include/x86_64-linux-gnu/sys/socket.h:117:47: note: expected ‘socklen_t * restrict’ {aka ‘unsigned int * restrict’} but argument is of type ‘int *’
  117 |                         socklen_t *__restrict __len) __THROW;
      |                         ~~~~~~~~~~~~~~~~~~~~~~^~~~~
```

At first glance, this is a simple fix. Here's the problematic `size` definition:

```c
int size;
```

I can change it like so:

```c
socklen_t size;
```

And that makes everything copacetic _for now_. But down the road, I will hit a snag. Here's the declaration for `getsockname` in Windows `winsock.h`:

```c
int getsockname(
  [in]      SOCKET   s,
  [out]     sockaddr *name,
  [in, out] int      *namelen
);
```

See how `int` has reterned to muck things up for us? I won't do anything about that, right now, but I will need to address it when I port to Windows.

For now, changing that one line actually fixed a host of warnings, leaving me with only one remaining for `comm.c`.

### Unused Variables

Here's the final warning for `comm.c`:

```
comm.c: In function ‘act_new’:
comm.c:2313:10: warning: variable ‘fColour’ set but not used [-Wunused-but-set-variable]
 2313 |     bool fColour = FALSE;
      |          ^~~~~~~
```

Compiler heuristics have come a long way since 1997. This might have flown back then because the variable _is_ accessed (line 2350), but never used. The  solution is to simply delete the variable and every reference to it.

And with that, `comm.c` is done.

### New Tools to Handle `sizeof()`

The next error, in `db.c` is related to `sizeof()`:

```
db.c: In function ‘do_dump’:
db.c:2556:40: warning: format ‘%d’ expects argument of type ‘int’, but argument 4 has type ‘long unsigned int’ [-Wformat=]
 2556 |     fprintf(fp, "MobProt        %4d (%8d bytes)\n", top_mob_index,
      |                                      ~~^
      |                                        |
      |                                        int
      |                                      %8ld
 2557 |             top_mob_index * (sizeof(*pMobIndex)));
      |             ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
      |                           |
      |                           long unsigned int
```

This being legacy C, there is an assumption that all sizes fit in an `int`, so that is what `sizeof()` returned. With the introduction of 64-bit computing, however, `sizeof()` needed more flexibility. The C90 standard fixed this by introducing `size_t`, an implementation-defined value intended to accomodate the actual, platform-specific range of valid sizes. `sizeof()` was changed to return this type instead of `int`.

But that means that using the `%d` (signed integer) specifier on systems where `size_t` is a 32-bit unsigned long integer is wrong. On those systems, you would have to use `%lu`. But that's problematic, too: what about systems that use `uint64_t` for `size_t`? They have to use `%llu` as the format specifier.

Thankfully, this problem was solved in C99 with the introduction of `%z` as a cross-platform format specifier for `size_t` for POSIX systems (and, as of 2013, Microsoft was dragged kicking and screaming into uniformity with everyone else).

The `printf` changes like so:

```c
    /* mobile prototypes */
    fprintf(fp, "MobProt	%4d (%8zu bytes)\n", top_mob_index,
            top_mob_index * (sizeof(*pMobIndex)));
```

I also make this change for all other `printf`'s taking `size_t` as an argument.

### In Which I Remember How Many Times I Got Hacked

There is one more warning from `save.c`:

```
save.c: In function ‘load_char_obj’:
save.c:650:33: warning: ‘%s’ directive writing up to 255 bytes into a region of size 90 [-Wformat-overflow=]
  650 |         sprintf(buf, "gzip -dfq %s", strsave);
      |                                 ^~   ~~~~~~~
In file included from /usr/include/stdio.h:894,
                 from merc.h:32,
                 from lookup.h:32,
                 from save.c:33:
/usr/include/x86_64-linux-gnu/bits/stdio2.h:38:10: note: ‘__builtin___sprintf_chk’ output between 11 and 266 bytes into a destination of size 100
   38 |   return __builtin___sprintf_chk (__s, __USE_FORTIFY_LEVEL - 1,
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   39 |                                   __glibc_objsize (__s), __fmt,
      |                                   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   40 |                                   __va_arg_pack ());
      |                                   ~~~~~~~~~~~~~~~~~
```

Here's the offending code:

```c
sprintf(buf, "gzip -dfq %s", strsave);
```

Where `strsave` and `buf` are defined thusly:

```c
char strsave[MAX_INPUT_LENGTH];
char buf[100];
```

Since `MAX_INPUT_LENGTH` is defined as `256`, this is a buffer overun waiting to happen. The first is, I need to widen the buffer:

```c
char buf[MAX_INPUT_LENGTH+12]; // To handle gzip cmd + strsave + \0
```

The second issue is wider: ROM performs almost no bounds checking on any string handlers. This will become an issue when I port to Windows, because MSVC _is simply not having it_. Not at all.

But for now, I will call this "fixed" and move on.

### It Builds Without Warnings!

No, it doesn't.

The ROM Makefile, unfortunately, doesn't have a "clean" setting, so I need to add it to `Makefile`:

```
clean:
	rm -f rom
	rm -f *.o
```

> NOTE: You _must_ use tabs in `Makefile`. It will complain about spaces. It is one of the many reasons I will be ditching `make` from this project very soon.

I then run the new option to clear up the binaries for a fresh run:

```
make clean && make
```

Now I have a whole lot more warnings to clean up. Previous files that had only warnings finished compiling and produced object files, so they never needed to be rebuilt. Cleaning the binaries made the compiler look at them again.

But I'm another step closer to finished, so there's that.

### Ambiguous Nesting

The next warning is common throughout the project:

```
act_info.c: In function ‘do_look’:
act_info.c:965:16: warning: suggest explicit braces to avoid ambiguous ‘else’ [-Wdangling-else]
  965 |             if (pdesc != NULL)
      |                ^
```

Here's the code in question from `act_info.c`:

```c
for (obj = ch->carrying; obj != NULL; obj = obj->next_content) {
    if (can_see_obj(ch, obj)) { /* player can see object */
        pdesc = get_extra_descr(arg3, obj->extra_descr);
        if (pdesc != NULL)
            if (++count == number) {
                send_to_char(pdesc, ch);
                return;
            }
            else
                continue;

        pdesc = get_extra_descr(arg3, obj->pIndexData->extra_descr);
            ...
```

Do you see the ambiguity? It's not a _syntactic_ ambiguity (the code is well-formed and correct) but a _visual_ ambiguity. In fact, this is one of those areas where "matching brace" style (which I usually detest) makes the problem apparent and obvious.

Every `if` here has an opening curly-brace but one:

```c
if (pdesc != NULL)
```

This makes it cover only the next line: itself also an `if` (but this time with an opening brace).

Now, the `else`; which `if` does it go with? Now, I happen to _know_ that `else` always matches the previous `if`, but it's easy to see how confusing it can be. And, to be sure, this very confusion leads to many errors (including many in ROM, itself).

Here's my rule for "bare" `if`s:

> Braces must be used with `if` unless the execution block for that `if` can fit on one line **and** its corresponding `else`'s (if there is one) can fit on one line, as well.

In other words, a real, actual "one-liner."

I'll fix this block like so:

```c
for (obj = ch->carrying; obj != NULL; obj = obj->next_content) {
    if (can_see_obj(ch, obj)) { 
        /* player can see object */
        pdesc = get_extra_descr(arg3, obj->extra_descr);
        if (pdesc != NULL) {
            if (++count == number) {
                send_to_char(pdesc, ch);
                return;
            }
            else {
                continue;
            }
        }

        pdesc = get_extra_descr(arg3, obj->pIndexData->extra_descr);
```

Note that I put `continue` in a braced block because the `if` corresponding to its `else` has a braced block. That's the rule I just established, and it makes the code more uniform. Whatever you do, do it consistently.

> Note that I also moved the comment to a new line. An open brace should be the _last_ thing on the line, so that it is not as visually obscured.

I won't notate all the places I make this change (there are many of them). But I will say that if you ever find yourself working in legacy C code, do yourself a favor and run it through `clang-format`. The brace and tab formatting in stock ROM is _awful_. Letting `clang-format` fix up the tabs visually establishes what blocks go with what logic structures. That's because a human might miss that hidden curly-brace, or elide over one that's missing, the computer will _not_.

Later on, there's this:

```c
if (count == 1)
    sprintf(buf, "You only see one %s here.\n\r", arg3);
else
    sprintf(buf, "You only see %d of those here.\n\r", count);
```

In my opinion, this is fine. But if either of those gets turned into a braced block, they both do.

Consistency!

But with one exception:

```c
    if (!str_cmp(arg1, "n") || !str_cmp(arg1, "north"))
        door = 0;
    else if (!str_cmp(arg1, "e") || !str_cmp(arg1, "east"))
        door = 1;
    else if (!str_cmp(arg1, "s") || !str_cmp(arg1, "south"))
        door = 2;
    else if (!str_cmp(arg1, "w") || !str_cmp(arg1, "west"))
        door = 3;
    else if (!str_cmp(arg1, "u") || !str_cmp(arg1, "up"))
        door = 4;
    else if (!str_cmp(arg1, "d") || !str_cmp(arg1, "down"))
        door = 5;
    else {
        send_to_char("You do not see that here.\n\r", ch);
        return;
    }
```

This is also from `act_info.c`, and I'm honestly okay with it. Because of the number of _consistent_ `if` statements and the exceptional (and obvious) nature of the `else`, I don't think there's any real danger.

Besides, programming is like jazz: once you know the rules inside and out, you are allowed to break them.

### Weird Switch

Later in `act_info.c`, in `do_who()` is this oddity:

```c
switch (wch->level) {
default:
    break;
    {
    case MAX_LEVEL - 0:
        class = "IMP";
        break;
    case MAX_LEVEL - 1:
        class = "CRE";
        break;
    case MAX_LEVEL - 2:
        class = "SUP";
        break;
    case MAX_LEVEL - 3:
        class = "DEI";
        break;
    case MAX_LEVEL - 4:
        class = "GOD";
        break;
    case MAX_LEVEL - 5:
        class = "IMM";
        break;
    case MAX_LEVEL - 6:
        class = "DEM";
        break;
    case MAX_LEVEL - 7:
        class = "ANG";
        break;
    case MAX_LEVEL - 8:
        class = "AVA";
        break;
    }
}
```

The _only_ reason this works is because C treats `case` statements like labels for `goto`. The fact that they are inside an enclosed scope does not prevent a jump because there is no stack manipulation happeningon the way there. But make so mistake; this is _bad_ code. Here's the fixed version:

```c
switch (wch->level) {
case MAX_LEVEL - 0: class = "IMP"; break;
case MAX_LEVEL - 1: class = "CRE"; break;
case MAX_LEVEL - 2: class = "SUP"; break;
case MAX_LEVEL - 3: class = "DEI"; break;
case MAX_LEVEL - 4: class = "GOD"; break;
case MAX_LEVEL - 5: class = "IMM"; break;
case MAX_LEVEL - 6: class = "DEM"; break;
case MAX_LEVEL - 7: class = "ANG"; break;
case MAX_LEVEL - 8: class = "AVA"; break;
default: break;
}
```

I took artistic liberty with the formatting because I love when repetitive code lines up like, and it happens most often in `switch` statements. Although, if I had my druthers, these strings would be in a table and you would look them up (because O(1) is king!).

Also: `default` goes last. Logically, it's the "fallback" scenario. It goes at the end.

### More Buffer Overflow

We have this doozy in `ban.c`:

```
ban.c: In function ‘ban_site’:
ban.c:148:32: warning: ‘    ’ directive writing 4 bytes into a region of size between 1 and 4596 [-Wformat-overflow=]
  148 |             sprintf(buf, "%-12s    %-3d  %-7s  %s\n\r", buf2, pban->level,
      |                                ^~~~
```

`ban_site` defines the buffers like so:

```c
char buf[MAX_STRING_LENGTH], buf2[MAX_STRING_LENGTH];
char arg1[MAX_INPUT_LENGTH], arg2[MAX_INPUT_LENGTH];
```

And here's the offending code:

```c
sprintf(buf2, "%s%s%s",
      IS_SET(pban->ban_flags, BAN_PREFIX) ? "*" : "", pban->name,
      IS_SET(pban->ban_flags, BAN_SUFFIX) ? "*" : "");
sprintf(buf, "%-12s    %-3d  %-7s  %s\n\r", buf2, pban->level,
      IS_SET(pban->ban_flags, BAN_NEWBIES)  ? "newbies"
      : IS_SET(pban->ban_flags, BAN_PERMIT) ? "permit"
      : IS_SET(pban->ban_flags, BAN_ALL)    ? "all"
                                          : "",
      IS_SET(pban->ban_flags, BAN_PERMANENT) ? "perm" : "temp");
add_buf(buffer, buf);
```

First, that code is an unholy abomination of nested tertiary logic. But that's not my problem, right now.

The code wants to write (left-justified) up to twelve ASCII string bytes from `buf2`; but `buf2` is a `char[4596]`. It's a buffer overflow waiting to happen. But _not_ because `buf2` is too _big_; the format specifier says _at most_ 12 characters. But what if `buf2` has _less_ than 12 characters? 

Here's the solution:

```c
sprintf(buf, "%-1.12s    %-3d  %-7s  %s\n\r", buf2, pban->level,
```

So, the format specifier `%-12s` means "exactly 12 characters" while `%-1.12s` means "between 1 and 12 characters". This fixes my immediate problem. When I move to Windows, however, this won't cut it. This function could potentially blow the stack, and MSVC will squawk about it. But I'll get to that, later.

### Undefined Behavior

In `handler.c` I find my first, real "bug":

```
handler.c: In function ‘affect_join’:
handler.c:1376:24: warning: operation on ‘paf->level’ may be undefined [-Wsequence-point]
 1376 |             paf->level = (paf->level += paf_old->level) / 2;
      |             ~~~~~~~~~~~^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
```

It's a bug because it's Undefined Behavior (UB). That means there is no specification for how this should even _work_. But we know that whatever it's tryin to do, it's unnecessary: it's assigning the end result to `paf->level`, overwriting whatever value `paf->level += paf_old->level` puts in there (and that's _if_ I'm right about the order of operations; so at best it's a NOOP, and at worst it's putting an unpredictable value in `paf->level`).

The context gives me a clue as to intent:

```c
/*
 * Add or enhance an affect.
 */
void affect_join(CHAR_DATA* ch, AFFECT_DATA* paf)
{
    AFFECT_DATA* paf_old;
    bool found;

    found = FALSE;
    for (paf_old = ch->affected; paf_old != NULL; paf_old = paf_old->next) {
        if (paf_old->type == paf->type) {
            paf->level = (paf->level += paf_old->level) / 2;
            paf->duration += paf_old->duration;
            paf->modifier += paf_old->modifier;
            affect_remove(ch, paf_old);
            break;
        }
    }

    affect_to_char(ch, paf);
    return;
}
```

So if it finds an "affect" (think status affect like "stunned", "blind", or "on fire") already in place, instead of adding it again, it replaces it with... the average? Something isn't right, here. Clearly, the intent was to make it _stronger_ if it already exists; but this could make the affect _weaker_ if the new level is less than the affect already there.

Effectively, this is what the code is without the spurious assignment in the middle:

```c
paf->level = (paf->level + paf_old->level) / 2; // The average
```

That means that if, say, you as a player attempt to poison an already poisoned monster, and your affect level is lower than what's already on the monster, you are _helping_ it instead of _hurting_ it. That makes no sense, and is obviously a bug. But fixing it will change behavior, and I don't want to do that, right now. So instead I'll fix the warning, and leave myself a note:

```c
// TODO: Don't just take the average; figure out how to increase level the
//       right way.
paf->level = (paf->level + paf_old->level) / 2;
```

### Even More Buffer Overflow

I have one more warning in `handler.c`, in `all_color()`:

```
handler.c: In function ‘all_colour’:
handler.c:3078:46: warning: ‘%s’ directive writing up to 99 bytes into a region of size 73 [-Wformat-overflow=]
 3078 |     sprintf(buf, "All Colour settings set to %s.\n\r", buf2);
      |                                              ^~        ~~~~
In file included from /usr/include/stdio.h:894,
                 from merc.h:32,
                 from interp.h:32,
                 from handler.c:33:
/usr/include/x86_64-linux-gnu/bits/stdio2.h:38:10: note: ‘__builtin___sprintf_chk’ output between 31 and 130 bytes into a destination of size 100
   38 |   return __builtin___sprintf_chk (__s, __USE_FORTIFY_LEVEL - 1,
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   39 |                                   __glibc_objsize (__s), __fmt,
      |                                   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   40 |                                   __va_arg_pack ());
      |                                   ~~~~~~~~~~~~~~~~~
```

This one is fairly straight forward, given that `sprintf()`, and the buffer definition of this:

```c
char buf[100];
char buf2[100];
```

I know what values these _need_ to be, because the warning itself tells me:

```c
char buf[132];
char buf2[100];
```

I added an extra two bytes so that it plays nice with the 4-byte boundary the compiler was going to give me, anyway.

> <p>Fun fact: When you allocate memory, unless you specify packed data (which you should only do in embedded), the compiler will give you the next highest multiple of 4 bytes, if the amount requested wasn't, already.</p><p>It has to do with how data is sent from the motherboard bus to the on-die cache, and the cost of grabbing scanlines starting at off-boundary locations. The compiler is trying to help you by making sure all your memory allocations begin in places that are the easiest and fastest to read from (and write to).</p>

The final warning, in `magic.h`, in `say_spell()`, is another buffer overrun. But this one is a big more obtuse:

```
magic.c: In function ‘say_spell’:
magic.c:159:42: warning: ‘%s’ directive writing up to 4607 bytes into a region of size 4586 [-Wformat-overflow=]
  159 |     sprintf(buf2, "$n utters the words, '%s'.", buf);
      |                                          ^~     ~~~
In file included from /usr/include/stdio.h:894,
                 from merc.h:32,
                 from interp.h:32,
                 from magic.c:33:
/usr/include/x86_64-linux-gnu/bits/stdio2.h:38:10: note: ‘__builtin___sprintf_chk’ output between 25 and 4632 bytes into a destination of size 4608
   38 |   return __builtin___sprintf_chk (__s, __USE_FORTIFY_LEVEL - 1,
      |          ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   39 |                                   __glibc_objsize (__s), __fmt,
      |                                   ~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
   40 |                                   __va_arg_pack ());
      |                                   ~~~~~~~~~~~~~~~~~
```

`4607` is the `MAX_STRING_LENGTH`, and it's already way too big for the stack (MSVC will not be putting up with this, as I'll demonstrate later) so we will not be "padding" the target buffer with extra bytes.

Using what I related earlier about format specifiers, I have this solution:

```c
sprintf(buf2, "$n utters the words, '%1.50s'.", buf);
```

This allows between 1 and 50 characters to be used in casting spells. Now, I have the benefit of knowing that these semi-random strings aren't very long at all, so I don't have a problem risking cutting off displayed data that already looks weird and incomprehensible.

No one will know.

But this, it's never going to _happen_, so I'm not concerned at all about it.

### No More Warnings; Time to Test It!

Wait, nope... sorry. One small problem: linker errors!


### Multiple Definitions

Heres the first one:

```
/usr/bin/ld: note.o:/home/bfelger/rom/Rom24/src/note.c:55: multiple definition of `note_list'; db.o:/home/bfelger/rom/Rom24/src/db.c:83: first defined here
```

Let's take a look at each in turn. First, `note.c`:

```c
NOTE_DATA* note_list;
```

And then in `db.c`:

```c
NOTE_DATA* note_list;
```

Obviously they cannot both exist in global scope. I need to make one `extern`. I choose always to keep concretions as close to their domain as possible, so I mark the copy in db.c as `extern`:

```c
extern NOTE_DATA* note_list;
```

This one is a bit more ambiguous: 

```
/usr/bin/ld: recycle.o:/home/bfelger/rom/Rom24/src/recycle.c:42: multiple definition of `note_free'; db.o:/home/bfelger/rom/Rom24/src/db.c:76: first defined here
```

I don't like that this function it's not in `note.c`, but I'm not refactoring anything, yet. For now, I will bias toward the fact that freeing (in this case, to the recycle pool) of other objects happens in `recycle.h`, so I will mark `db.c`'s copy as `extern`.

I have one, final linker error:

```
/usr/bin/ld: act_info.o: in function `do_password':
/home/bfelger/rom/Rom24/src/act_info.c:2431: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/act_info.c:2446: undefined reference to `crypt'
/usr/bin/ld: comm.o: in function `nanny':
/home/bfelger/rom/Rom24/src/comm.c:1482: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/comm.c:1586: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/comm.c:1607: undefined reference to `crypt'
```

This one is dumb. I need to include `-lcrypt` in the linker in `Makefile`, but I can't do this:

```
NOCRYPT = 
C_FLAGS =  -Wall $(PROF) $(NOCRYPT)
L_FLAGS =  $(PROF) -lcrypt
```

That's because of the linker call:

```
rom: $(O_FILES)
    rm -f rom
    $(CC) $(L_FLAGS) -o rom $(O_FILES)
```

This puts the `-lcrypt` linker option near the beginning of the command, and GCC is kind of stupid in the way it handles linker precedence. Instead I need to make sure it's the last thing on the linker line, like so:

```
rom: $(O_FILES)
    rm -f rom
    $(CC) $(L_FLAGS) -o rom $(O_FILES) -lcrypt
```

And I get to see my handiwork:

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

No warnings, and no errors, and I have a `rom` executable in my `src` dir. Time to let 'er rip!

### Taking ROM For a Test Drive After 26 Years

ROM has to be started from the `area` directory like so:

```
cd ..
cd area
../src/rom 4000
```

And here's what we get:

```
Sat May  6 00:00:41 2023 :: [*****] BUG: Fix_exits: 10525:1 -> 10535:3 -> 10534.
Sat May  6 00:00:41 2023 :: [*****] BUG: Fix_exits: 3458:2 -> 3472:0 -> 10401.
Sat May  6 00:00:41 2023 :: [*****] BUG: Fix_exits: 8705:4 -> 8706:5 -> 8708.
Sat May  6 00:00:41 2023 :: [*****] BUG: Fix_exits: 8717:2 -> 8719:0 -> 8718.
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
Sat May  6 00:00:41 2023 :: ROM is ready to rock on port 4000.
```

A bunch of errors in the area files (_**BOOOOORING!!!**_) and those sweet, magical words: `ROM is ready to rock on port 4000.`

> Seriously, I have never seen ROM not run without those object errors on boot. I don't even think I've noticed any issues arising from them.

To log in, I need a telnet client. Unfortunately, the fuddyduddies at Microsoft and most Linux distros want to stop me from executing my God-given right to send my universal, catch-all password over the internet in _clear text_, so I will need to go out of my way to find such a client.

Thankfully, [Mudlet](https://www.mudlet.org/) is still under active development, and opens right out of the box.

I fire it up, and this is the result:

```
THIS IS A MUD BASED ON.....

                                ROM Version 2.4 beta

               Original DikuMUD by Hans Staerfeldt, Katja Nyboe,
               Tom Madsen, Michael Seifert, and Sebastian Hammer
               Based on MERC 2.1 code by Hatchet, Furey, and Kahn
               ROM 2.4 copyright (c) 1993-1998 Russ Taylor

By what name do you wish to be known? 
```

Hot diggity-dog! My mind already swirls with the possibility of all the things I wanted to code in ROM, but didn't have the talent to do all those decades go. But this is just the first step. I want to be 100% cross-compiler and cross-platform.

But that's for another day.

([Here is the code](https://github.com/bfelger/rom/tree/51a10fee23516ad44250bc652bae78f52460d942) with updates from this post.)

Copyright 2023, Brandon Felger