---
layout: default
title:  "Resurrecting ROM Pt. 6 - Compiling ROM With MSVC (Part 2)"
date:   2023-05-18 20:00:00 -0500
categories: mud C CMake MSVC
---

# Resurrecting ROM Pt. 6 &mdash; Compiling ROM With MSVC (Part 2)

This post picks up where [Part 5](pt-5-msvc) left off. [Here is the code](https://github.com/bfelger/rom/tree/70717079d2b4bb3a1e3ab1fd692e165ee8cf6e07) that resulted from the work.

I got ROM building and running without errors on MSVC on Windows. The task was made all the easier with the help of CMake. But my stated original goal was that it should compile on all platforms at my disposal (GCC, Clang, Cygwin, and MSVC) without errors _or warnings_.

Alas, I still have hundreds of those.

Thankfully, for the most part they fall into a handful of categories. The changes I will make in this post will be minor, but pervasive.

Also, just a _little bit_ tedious.

## The Low-Hanging Fruit

I want to start with the most common warnings, as clearing those will chop the number of warnings I have very quickly. 

### VT100 Escape Codes

Lope's ColoUr code uses VT100 escape sequences that look like this:

```c
/*
 * ColoUr stuff v2.0, by Lope.
 */
#define CLEAR              "\e[0m" /* Resets Colour	*/
#define C_RED              "\e[0;31m" /* Normal Colours	*/
#define C_GREEN            "\e[0;32m"
#define C_YELLOW           "\e[0;33m"
#define C_BLUE             "\e[0;34m"
#define C_MAGENTA          "\e[0;35m"
#define C_CYAN             "\e[0;36m"
#define C_WHITE            "\e[0;37m"
#define C_D_GREY           "\e[1;30m" /* Light Colors		*/
#define C_B_RED            "\e[1;31m"
#define C_B_GREEN          "\e[1;32m"
#define C_B_YELLOW         "\e[1;33m"
#define C_B_BLUE           "\e[1;34m"
#define C_B_MAGENTA        "\e[1;35m"
#define C_B_CYAN           "\e[1;36m"
#define C_B_WHITE          "\e[1;37m"
```

These codes are interpreted by telnet terminals (and local text consoles, or anything implementing VT100) as commands to manipulate the screen buffer of the terminal at the other end. You can use them to change cursor position, to clear the screen, or (as it is used here) to apply color to the text.

But there is an warning related to these codes:

```
Warning	C4129	'e': unrecognized character escape sequence	
```

Now, if you boot up ROM, log in, and type `colour on`, you will see that the compiler did in fact know how to compiler the escape sequence (`\e`). What the compiler is telling me here is that this is nonstandard. Strictly speaking, it is UB (undefined behavior). It's giving me the output I want, _but it doesn't have to_. And I don't really have a reasonable expectation of it always doing so.

To get rid of these warnings all at once, I globally replace "`\e[`" with "`\033[`". That's the octal number for the Escape character, and is the cross-platform way to represent VT100 codes.

### Format Specifier for `time_t`

I have several dozens of a variation of this:

```
note.c(175): warning C4477: 'fprintf' : format string '%ld' requires an argument of type 'long', but variadic argument 1 has type 'time_t'
  note.c(175): note: consider using '%lld' in the format string
  note.c(175): note: consider using '%Id' in the format string
  note.c(175): note: consider using '%I64d' in the format string
```

Here's the offending code:

```c
fprintf(fp, "Stamp   %ld\n", pnote->date_stamp);
```

This was correct in Linux, where `time_t` is backed by a `long int`. In MSVC, however, `time_t` is backed by `int64_t`. That requires a different format specifier.

To solve this issue, I add a `TIME_FMT` macro to `strings.h`:

```c
// Handle time_t printf specifiers
#ifdef _MSC_VER
#define TIME_FMT "%" PRIi64
#else
#define TIME_FMT "%ld"
#endif
```

> Isn't kind of a stretch to put this in `strings.h` just because it has to do with `printf()`?
>
> Yes. Yes, it is.

I reformat the offending code like so:

```c
fprintf(fp, "Stamp   "TIME_FORMAT"\n", pnote->date_stamp);
```

The compiler will concatenate these string literals at compile time.

### Retrofitting `size_t`

Many years ago, `int` was the universal assumed data type for memory sizes and string length. Today, however, that role belongs to `size_t`. POSIX functions like `strlen()` return it. GCC and Clang didn't so much mind ROM using `int`, but MSVC very much does.

So I have lot of warnings like this, from `do_description()` in `act_info.c`:

```
Warning	C4267 '=': conversion from 'size_t' to 'int', possible loss of data	C:\cygwin64\home\bfelg\rom\src\act_info.c 2187
```

Here's where that warning occurs:

```c
for (len = strlen(buf); len > 0; len--) {
```

Where `len` is declared as an `int` at the top of the scope. The simplest solution is to change its declaration to `size_t`, and that will take care of many cases. But if I look a little closer, I notice that `len` is only used as a counter for this single loop. So instead, I remove the declaration of `len` from the outer scope and move it into the loop itself (valid since C99):

```c
for (size_t len = strlen(buf); len > 0; len--) {
```

> A point about pre-and-post-inc/decrement: historically, the prefix version (`--len`) would have been faster, as the original value doesn't need to be preserved to be used as an rval. However, all modern C compilers (or, rather, any worth using) optimize a bare postfix (`len--`) that is not used as an rval into the same exact target code.

In other areas, I need to change function signatures:

```c
void* alloc_mem(size_t sMem)
```

This change, in turn, spiders out, necessitating more changes to call sites.

In general, any variable or function argument named `len`, `size`, or `sMem` needs to be a `size_t` (except for notable, obvious exceptions like the `size` argument for `dice()`; that's the _die-type_ to roll).

Wherever former `int`s-turned-`size_t`'s are used in `printf()` or anything else that requires a format specifier, I change "`%d`" to "`%zu`".

There are some areas I don't want to futz with, like `fread_string()` in `db.c`:

```c
iHash = UMIN(MAX_KEY_HASH - 1, plast - 1 - top_string);
```

Here, `plast` and `top_string` are both `char*` strings. This results in what is essentially a `ptrdiff_t`. That delta is from within the same block of memory up to `MAX_STRING`. Since that amount is exceedingly small (compared to `int`'s positive range), I feel no compunction about just casting it:

```c
iHash = (int)UMIN(MAX_KEY_HASH - 1, plast - 1 - top_string);
```

Now, in some areas I can avoid the cast and responsibly use `ptrdiff_t` the way it was intended, like in `descriptor_data` in `merc.h`:

```c
    size_t outsize;
    ptrdiff_t outtop;
```

To make this work in GCC/Clang, however, I also have to add this to the top of the header:

```c
#ifndef _MSC_VER
#include <stddef.h>
#endif
```

Another consideration is function return types, like `colour()` in `comm.c`:

```c
int colour(char type, CHAR_DATA* ch, char* string)
```

But here's the last line of the function:

```c
    return strlen(code);
```

> This code was originally parameterized (`return (strlen(code));`). I will remove them everywhere I see them.
>
> To thine own self be true.

Since `strlen()` returns `size_t`, I need to change `colour()`'s signature to match, and then accommodate that everywhere `colour()` is called.

### Fixing up `bug()`

ROM often calls `bug()` (in `db.c`) with different types of arguments and different sizes. It only takes `int`, however, which is problematic for us: we need it to _also_ now take `size_t`. I refuse to duplicate code, so instead I will do what should have been done in the first place, and give `bug()` the same variadic flexibility as `printf()`.

First, I include `stdarg.h` in `db.c`:

```c
#include <stdarg.h>
```

Then I change `bug()`'s signature:

```c
void bug(const char* fmt, ...)
```

Then I add var-arg handling to the function, itself:

```c
void bug(const char* fmt, ...)
{
    char buf[MAX_STRING_LENGTH];

    va_list args;
    va_start(args, fmt);
```

All that now remains to change `sprintf()` to `vsprintf()` and pass in the scarfed args:

```c
    vsprintf(buf + strlen(buf), fmt, args);
```

Now this works:

```c
    bug("Alloc_mem: size %zu too large.", sMem); // sMem is size_t
```

But there's solid value add! We can _also_ now do this:

```c
    bug("Alloc_mem: size %zu too large; max is %zu.", sMem, MAX_MEM_SIZE);
```

> That's just an example; there is no `MAX_MEM_SIZE`. But that would be an easy shortcut in `Alloc_mem()` to avoid looping through the buckets, wouldn't it?

### Casting `time_t` to `int`

I have a few items like this:

```
Warning	C4244 '=': conversion from 'time_t' to 'long', possible loss of data
C:\cygwin64\home\bfelg\rom\src\db.c 238	
```

Here is the troubled code, in `boot_db()` in `db.c`:

```c
long lhour, lday, lmonth;

lhour = (current_time - 650336715) / (PULSE_TICK / PULSE_PER_SECOND);
time_info.hour = lhour % 24;
lday = lhour / 24;
time_info.day = lday % 35;
lmonth = lday / 35;
time_info.month = lmonth % 17;
time_info.year = lmonth / 17;
```

That magic number UNIX epoch time for Saturday, August 11, 1990 1:05:15 AM GMT. It grabs the time elapsed from that moment and begins chopping it up to feed into the game's weather updater.

This is another situation where something less than the full 64-bit type is good enough, especially since it's delta'd from a (what was at the time) a relatively recent UNIX epoch. Therefore, I'm perfectly comfortable with casting to `long` after doing all division operations on the input `time_t`. But to be safe, I update that elapsed time reference value to this morning. This makes the resultant value two bits smaller, and maybe buys me time to limp into to 2050.

Here's the final code, after some rearranging:

```c
long lhour = (long)((current_time - 1684281600) 
    / (PULSE_TICK / PULSE_PER_SECOND));
long lday = lhour / 24;
long lmonth = lday / 35;

time_info.hour = lhour % 24;
time_info.day = lday % 35;
time_info.month = lmonth % 17;
time_info.year = lmonth / 17;
```

## The One-Offs

Now I have ROM down to 32 warnings and 56 messages. The low-hanging fruit is gone, and now each warning is local in context and requires a fitted solution.

### Schr&ouml;dinger's Double-Cast

Take this line from `create_mobile()` in `db.c`:

```c
mob->max_hit *= .9;
```

If `mod->max_hit` is `int16_t`, where does casting occur? Is `max_hit` cast to `double` and then back to `int16_t`? Is `0.9` cast to `uint16_t` (becoming zero) before multiplication?

I'm sure the C Standard has an answer, but I would much prefer to explicitly cast to make the clear, common-sense intention plain:

```c
mob->max_hit = (int16_t)((double)mob->max_hit * 0.9);
```

Here, I chose to delay rounding until the very last assignment so as to not compound the loss of precision.

### Short Flags

There's a function to read flags (those bit-vectors defined in `merc.h`) from player and area files. Status affect flags need a 32-bit `long`, and that is what `read_flag()` returns. Ban flags, however, being very limited in number, are stored in a `int16_t` (in the original code, it was a `sh_int`).

In this case, it's perfectly safe to cast `long` away:

```c
pban->ban_flags = (int16_t)fread_flag(fp);
```

### Deprecated POSIX

This warning is a simple enough fix:

```
ban.c(62): warning C4996: 'unlink': The POSIX name for this item is deprecated. Instead, use the ISO C and C++ conformant name: _unlink. See online help for details.
```

This also appears in another file, so I will add an `#else` to the MSVC exclusion guard for both `ban.c` and `act_comm.c`:

```c
#ifndef _MSC_VER 
#include <sys/time.h>
#include <unistd.h>
#else
#define unlink _unlink
#endif
```

This needs to be _after_ the include files, so that it shadows the definition of `unlink` from them.

### Late-Initialized Pointers

```
Warning C6011 Dereferencing NULL pointer 'ban_last'.C:\cygwin64\home\bfelg\rom\src\ban.c 92
```

Here's the problem in `load_bans()` in `bans.c`:

```c
    if (ban_list == NULL)
        ban_list = pban;
    else
        ban_last->next = pban;
    ban_last = pban;
```

Here is `ban_last`'s definition (after consolidating declaration and initialization):

```c
BAN_DATA* ban_last = NULL;
```

Take a look at that first chunk; the problem chunk. Do you see what it's doing? It's building a linked list. But the compiler is confused because it doesn't realize that if ban_list is _not_ NULL, then it's a later iteration where `ban_last` was set. I add another NOOP conditional to make things clear to the compiler:

```c
    if (ban_list == NULL)
        ban_list = pban;
    else if (ban_last)
        ban_last->next = pban;
    ban_last = pban;
```

I also fix similar issues with `load_helps()` and `load_resets()` in `db.c`.

There's also a spurious NULL dereference warning in `load_shops()`  at the `alloc_perm()` call that I fix like so:

```c
    if ((pShop = (SHOP_DATA*)alloc_perm(sizeof(*pShop))) == NULL) {
        bug("load_shops: Failed to create shops.");
        exit(1);
    }
```

The test for NULL and subsequent warning are actually a NOOP; `alloc_perm()` tests for NULL in itself, and kills the program. This is just to mollify the compiler in an code-consistent way.

`pReset` in `load_resets()` needs similar treatment.

### Warnings in PCG

There is a trio of warnings (two different codes) that originate in some of the black-magic math at the heart of PCG, the PRNG implemented in `pcg_basic.c`. Given that it was written, distributed, and used by people far, far smarter than I am, I cannot presume to "fix" code that is inscrutable to me.

Therefore _those_ warnings I will ignore, by putting this at the top of `pcg_basic.c`:

```c
#pragma warning( disable : 4146 4244 )
```

GCC and Clang won't like this, but I can tell them to not to complain by adding this flag in `CMakeLists.txt`:

```cmake
if (CMAKE_C_COMPILER_ID STREQUAL "GNU" 
    OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-Wno-unknown-pragmas)
endif()
```

### Printing a Socket?

This one is in `do_sockets()` in `act_info.c`:

```
Warning C6328 Size mismatch: 'unsigned __int64' passed as _Param_(3) when 'int' is required in call to 'sprintf'.
C:\cygwin64\home\bfelg\rom\src\act_wiz.c 3513
```

Here is the call site:

```c
sprintf(buf + strlen(buf), "[%3d %2d] %s@%s\n\r", d->descriptor,
        d->connected,
        d->original    ? d->original->name
        : d->character ? d->character->name
                        : "(none)",
        d->host);
```

ROM is trying to print the socket (a scalar number) as an `int` (`%d`). But `d->descriptor` will be different on different platforms: `long int` (`%ld`) on Linux, and `long long int` (`%lld`) on MSVC.

But here's the thing about sockets as scalar integers: if you debug the program a hundred times, you're never going to see them surpass three digits or so. MSVC insists on `__Int64` for `SOCKET`, but it does not _need_ that.

So here is my fix:

```c
sprintf(buf + strlen(buf), "[%3ld %2d] %s@%s\n\r", (long)d->descriptor,
```

### Inconsistent Linkage of `rename()`

Here is the last warning:

```
Warning	C4273 'rename': inconsistent dll linkage
C:\cygwin64\home\bfelg\rom\src\save.c 45
```

The call comes from the end of `save_char_obj()` in `save.c`:

```c
rename(TEMP_FILE, strsave);
```

"Inconsistent DLL linkage" means that the function signature declared in code doesn't match the function signature declared in code.

It was pulled in from an include, right?

```c
int rename(const char* oldfname, const char* newfname);
```

That's from the top of `save.c`. By explicitly defining `rename()` it shadowed the actual definition from `stdio.h`.

Don't manually define library functions. Not even once.

But there's still a problem with it: it returns an error code, and this error code is not being used. I fix by adding error reporting:

```c
if (rename(TEMP_FILE, strsave) < 0) {
    bug("Save_char_obj: failed to rename file %s to %s.\n", TEMP_FILE, strsave);
}
```

Notice here, the first application of the new, improved variadic `bug()` function. It's taking strings! And more than one, to boot.

### Uninitialized Local Variables

This isn't a warning, per se; it's a message. But it will be impacted by changes I make for  warning, so I will go ahead and address it before I move on.

This message comes from the code-linter, and it repeated 41 times, all in different places (but mostly in `comm.c`):

```
Message lnt-uninitialized-local Local variable is not initialized.
C:\cygwin64\home\bfelg\rom\src\comm.c 345
```

Scalar values are a trivial case:

```c
size_t nWrite = 0;
```

Note that a `char` is really just a small integer:

```c
char letter = 0;
```

But in each case, I will check to see if it can be declared and initialized with its first actual usage (allowed since C99).

And of course, pointers are initialized to `NULL`:

```c
DESCRIPTOR_DATA* d_old = NULL;
DESCRIPTOR_DATA* d_next = NULL;
```

Now, what about aggregates, like `struct`'s?

```c
fd_set in_set;
fd_set out_set;
fd_set exc_set;
```

Here's how you initialize a `struct` to all zeros:

```c
fd_set in_set = { 0 };
fd_set out_set = { 0 };
fd_set exc_set = { 0 };
```

This also works with `union`'s:

```c
union {
    char* pc;
    char rgc[sizeof(char*)];
} u1 = { 0 };
```

Initializing `char[]` string buffers is a little more intuitive:

```c
char buf[MAX_STRING_LENGTH] = "";
char buf2[MAX_STRING_LENGTH] = "";
char buffer[MAX_STRING_LENGTH * 2] = "";
```

### Possible Stack Overflow

There are 4 warnings in `comm.c` that go like this:

```
Warning C6262 Function uses '18792' bytes of stack. Consider moving some data to heap.
C:\cygwin64\home\bfelg\rom\src\comm.c 907
```

Here is one of the functions that is a problem child in `comm.c`, and it should look familiar:

```c
void bust_a_prompt(CHAR_DATA* ch)
{
    char buf[MAX_STRING_LENGTH] = "";
    char buf2[MAX_STRING_LENGTH] = "";
    char buffer[MAX_STRING_LENGTH * 2] = "";
```

Since `MAX_STRING_LENGTH` is 4608 bytes, these three fields _by themselves_ use almost 18.5k bytes. That's doable; but it's also _ludicrous_.

And keep in mind that many of these functions call other functions that _also_ use up string buffers. 

There needs to be a new way to do this that doesn't put the stack in danger. Thankfully, a solution is already in place in ROM: `BUFFER` (aka `buffer_type`). It creates buffers from ROM's own memory allocator and serves them up. At the moment, it is only used for output buffers. I will extend it for use in transient buffers, as well.

First, I need to add a couple macros to `recycle.h` to help smooth the transition to using `BUFFER`. Here is a helper macro for `recycle.h` that will pay off in a short while:

```c
#define INIT_BUF(b, sz) BUFFER* b = new_buf_size(sz)
#define BUF(b) (b->string)
```

Then, in `bust_a_prompt()` in `comm.c`, I replace the buffer definitions like so:

```c
    INIT_BUF(buf, MAX_STRING_LENGTH);
    INIT_BUF(buf2, MAX_STRING_LENGTH);
    INIT_BUF(buffer, MAX_STRING_LENGTH * 2);
```

I make sure to add a clean-up section at the end. **If I don't do this, then memory will leak.** I put a label above it because I need to be able to go straight to clean-up from anywhere in the function:

```c
bust_a_prompt_cleanup:
    free_buf(buf);
    free_buf(buf2);
    free_buf(buffer);
```

I then replace references to each buffer variable with the handy-dandy macro I made earlier:

```c
case 'x':
    sprintf(BUF(buf2), "%d", ch->exp);
    i = BUF(buf2);
    break;
```

To prevent leaks, I comb through the code of the function and replace `return` with a jump to clean-up:

```c
    if (IS_SET(ch->comm, COMM_AFK)) {
        send_to_char("{p<AFK>{x ", ch);
        goto bust_a_prompt_cleanup;
    }
```

> Yeah, it's `goto`. It's the tool for this job, and there's no better one. 
>
> But if it makes you feel any better, feel free to read Dijkstra's "Use of 'Go-To' Considered Harmful" and sleep well knowing that this is not the kind of code he was complaining about.

Note that there is no need for the label if there is no early exit from the function (in fact, the label in that case would generate its own warning).

> Oh, and I totally changed the buffer names in my actual code to `temp1`-`3`, because `buf`, `buf2`, and `buffer`? Terrible. I'll go back later and replace `temp` with what the buffer is actually used for.

Now, there is a problem with the way I'm using `BUFFER`: grabbing `MAX_STRING_LENGTH*4` will crash the app. Here's why (in `recycle.c`):

```c
/* buffer sizes */
const int buf_size[MAX_BUF_LIST]
    = {16, 32, 64, 128, 256, 1024, 2048, 4096, 8192, 16384};
```

At is turns out, `MAX_STRING_LENGTH*4` is larger than the largest allowed buffer. To fix this, I am expanding `buf_size` (and therefore also the value of `MAX_BUF_LIST` in `merc.h`) by 3:

```c
/* buffer sizes */
const int buf_size[MAX_BUF_LIST]
    = {16, 32, 64, 128, 256, 1024, 2048, 4096, MAX_STRING_LENGTH, 8192, 
        MAX_STRING_LENGTH*2, 16384, MAX_STRING_LENGTH*4};
```

The smaller two new values fit in the larger buckets, but I don't _want_ them too. That is too much wasted space for my liking.

### Zero-Termination

Here's the last warning, `write_to_buffer()` in `comm.c`:

```
Warning C6053 The prior call to 'strncpy' might not zero-terminate string 'd->outbuf'.
C:\cygwin64\home\bfelg\rom\src\comm.c 1104
```

Here's the code, in context:

```c
    /*
     * Expand the buffer as needed.
     */
    while (d->outtop + length >= d->outsize) {
        char* outbuf;

        if (d->outsize >= 32000) {
            bug("Buffer overflow. Closing.\n\r", 0);
            close_socket(d);
            return;
        }
        outbuf = alloc_mem(2 * d->outsize);
        strncpy(outbuf, d->outbuf, d->outtop);
        free_mem(d->outbuf, d->outsize);
        d->outbuf = outbuf;
        d->outsize *= 2;
    }
```

This line may be unnecessary, but it's a good idea:

```c
        strncpy(outbuf, d->outbuf, d->outtop);
        outbuf[d->outtop] = '\0';
```

## Compiler Messages

Now that all the warnings are gone, I want to tackle messages (of which there are 42). These aren't even warnings; the compiler wants to draw my attention to things that may, or may not be a problem. Some of these things will be mistakes, anti-patterns, or false positives.

### Possible Integer Overflow

In `recycle.c`, the functions `get_pc_id()`, `get_mob_id()`, and their respective backing values are `long`. But the id's, themselves, have ternary logic with `time_t`.

```c
long val = (long)((current_time <= last_pc_id) ? last_pc_id + 1 : current_time);
```

To cheese it, I upcast `last_pc_id` inside the ternary:

```c
long val = (long)((current_time <= last_pc_id) 
            ? (time_t)last_pc_id + 1 
            : current_time);
```

### `NULL` File Pointer

Back to `save_char_obj()` in `save.c`:

```
Warning C6387 'fp' could be '0'.
C:\cygwin64\home\bfelg\rom\src\save.c 129
```

The line number on these is not quite correct. The actual problem is in the greater context:

```c
    if ((fp = fopen(TEMP_FILE, "w")) == NULL) {
        bug("Save_char_obj: fopen", 0);
        perror(strsave);
    }
    else {
        fwrite_char(ch, fp);
        if (ch->carrying != NULL) fwrite_obj(ch, ch->carrying, fp, 0);
        /* save the pets */
        if (ch->pet != NULL && ch->pet->in_room == ch->in_room)
            fwrite_pet(ch->pet, fp);
        fprintf(fp, "#END\n");
    }
    fclose(fp);
```

Is says the problem is the first line; but it's actually the _last_.

`fclose()` runs unconditionally; it should be one line higher, as the last statement in the `else` block.

## Coda

And with that, ROM now builds, error _and_ warning-free, on all four of my targeted platforms. I now have my "baseline": ROM Resurrected.

Future noodlings in this code base will be built on the work I did here.

([Here is the code](https://github.com/bfelger/rom/tree/ac968f668dd3f5c96eee55d6e855641d6a8ba496) with updates from this post.)

Copyright 2023, Brandon Felger
