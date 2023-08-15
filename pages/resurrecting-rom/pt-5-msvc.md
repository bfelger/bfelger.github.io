---
layout: default
title:  "Resurrecting ROM Pt. 5 - Compiling ROM With MSVC (Part 1)"
date:   2023-05-16 20:00:00 -0500
categories: mud C CMake MSVC
---

# Resurrecting ROM Pt. 5 &mdash; Compiling ROM With MSVC (Part 1)

This post picks up where [Part 4](pt-4-cmake) left off. [Here is the code](https://github.com/bfelger/Mud98/tree/f03fa77b6e4779dc2e03b0b88c93bbe7d2cc0c3b) that resulted from that work.

Previously, I made every attempt to squash all errors (and warnings) on each platform in one go. To do that on Windows would balloon this post, and create so many deltas in the git check-in that it would be utterly unintelligible. 

Therefore, I will keep my goal for this "Part 1" squarely on getting ROM to _run_ under MSVC for Windows. I will fixing only errors, and leave all the warnings and other compiler messages for "Part 2" (because the stakes will be lower).

## Opening the Folder with Visual Studio

First, I use Visual Studio's "Open Folder" option, and select the `src` folder.

I immediately see this in the Output window:

```
1> [CMake] -- The C compiler identification is MSVC 19.35.32217.1
1> [CMake] -- The CXX compiler identification is MSVC 19.35.32217.1
1> [CMake] -- Detecting C compiler ABI info
1> [CMake] -- Detecting C compiler ABI info - done
1> [CMake] -- Check for working C compiler: C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.35.32215/bin/Hostx64/x64/cl.exe - skipped
1> [CMake] -- Detecting C compile features
1> [CMake] -- Detecting C compile features - done
1> [CMake] -- Detecting CXX compiler ABI info
1> [CMake] -- Detecting CXX compiler ABI info - done
1> [CMake] -- Check for working CXX compiler: C:/Program Files/Microsoft Visual Studio/2022/Community/VC/Tools/MSVC/14.35.32215/bin/Hostx64/x64/cl.exe - skipped
1> [CMake] -- Detecting CXX compile features
1> [CMake] -- Detecting CXX compile features - done
1> [CMake] -- Configuring done
1> [CMake] -- Generating done
1> [CMake] -- Build files have been written to: C:/cygwin64/home/bfelg/rom/src/out/build/x64-Debug
```

Because Ninja is MSVC's prefer CMake generation tool, it should look familiar. One difference is that it chooses to put all the binaries in the `out` folder, so I will add that to my `.gitignore` file.

Right now, we only have one configuration: Debug. Another thing I notice is that the compiler flags I set in `CMakeLists.txt` were not honored:

```cmake
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_compile_options(/W4 /WX /std:c17)
```

The fix for this turns out to be quite simple:

```cmake
if (MSVC)
    add_compile_options(/W4 /WX /std:c17)
```

It mismatches the checks for GCC and Clang, but I'm okay with that, as long as we get running. Theoretically speaking, changing `CMakeLists.txt` is something that should be fairly rare.

## Building for the First Time (or Trying to, Anyway)

After selecting `Build` \| `Build All` I let out a little shriek: 22 errors, and 772 warnings. But then I look a little closer and I see a lot of "unused parameter" warnings, and all of them treated as errors. 

As it turns out, my MSVC warning settings are much more aggressive than I chose for GCC/Clang. Effectively, it's the equivalent of `-Wall -Wextra -pedantic -Werror`. I mentions before that I would like to get there, but not quite yet.

Instead I choose to dial back my settings, just to get us off the ground:

```cmake
    add_compile_options(/W3 /std:c17)
```

That takes me down to 16 errors, and 316 warnings. That's doable.

### Missing `sys/time.h` and Other UNIX Headers

Here's the first error, in `act_move.c`:

```
Cannot open include file: 'sys/time.h': No such file or directory
C:\cygwin64\home\bfelg\rom\src\act_move.c 33 
```

> Note that I am not running in the Cygwin environment; that just so happens to be where my code is sitting, at the moment.

This is a UNIX-specific header, so I gate it behind MSVC:

```c
#ifndef _MSC_VER 
#include <sys/time.h>
#endif
```

That's an odd macro. Why not use `WIN32` or `WIN64`, which are also predefined macros in MSVC? Well, there has historically been debate on both Cygwin and GCC mailing lists as to whether `WINXX` is a _compiler_ or _platform_ indicator. And the decision on whether those predefined macros should be set differ between compilers hosted on Cygwin. In my mind (and I could be wrong) there is a real danger of polluting Cygwin builds with MSVC-targeted code if I use those macros.

On the other hand, `_MSC_VER` is a numerical macro that is _always_ and _only_ set in MSVC: it's the version number of the current MSVC compiler.

Since `sys/time.h` is included in a lot of files, I need to change them all to match the guarded version above. Unsurprisingly, we need to give `unistd.h` (I mean, it says "UNIX Standard" right in the name) the same treatment, as well as `sys/resource.h`.

In `comm.c`, I have my largest list of missing headers, yet:

```c
#ifndef _MSC_VER 
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>
#endif
```

`comm.c` is there I'm going to spend most of my rewriting time. I have to replace the entirety of Berkeley Sockets with the WinSock library; and it is by no means a drop-in replacement.

### Culling `crypt.h`

I also get an error here:

```c
#ifndef _XOPEN_CRYPT
#include <crypt.h>
#endif
```

```
Cannot open include file: 'crypt.h': No such file or directory
```

Because of _course_. To add a second condition, I have to change the format of the `#if`:

```c
#if !defined(_XOPEN_CRYPT) && !defined(_MSC_VER)
#include <crypt.h>
#endif
```

Windows does not have `crypt()`. It _does_ have a Crypto API, but it's a "real" crypto implementation. And that's more complicated than I want to get into, right now. When the time comes, I will replace `-lcrypt` on all platforms with a cross-platform solution.

So, for now, I will `CMakeLists.txt` to add `-D NOCRYPT` to it's definitions:

```cmake
    add_definitions(-D NOCRYPT)
```

And then I add a guard around the library link:

```cmake
if (NOT MSVC)
    target_link_libraries(rom crypt)
endif()
```

We're going to be "riding dirty" with player passwords under MSVC for a bit until we get a real solution in place.

## It Looks Like We Aren't in UNIX Anymore, Toto

The next group of errors to fix all come from `comm.c`, and have to do with the UNIX network code, which is based on Berkely Sockets. When Merc and Diku were first written, Mac and DOS had no network stack. Today Mac is a full-fledged POSIX system, while Windows is... I'll call it "POSIX-adjacent". It is similar _enough_ that I can graft the Windows network library (called `WinSock`) into the existing code.

But doing that is going to take some time and research.

### Adding WinSock

This is the next error, on `struct timeval` in `comm.c`:

```
Error (active) E0070 incomplete type is not allowed rom.exe - x64-Debug
C:\cygwin64\home\bfelg\rom\src\comm.c 118
```

We moved the reference to the UNIX header that contains the definition for `struct timeval` behind a `_MSC_VER` guard. To replace it, we need to add `winsock.h`:

```c
#ifdef _MSC_VER 
#include <winsock.h>
#else
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>
#endif
```

### Signal Pipes

As I mentioned, `comm.c` is going to be a problem child for me. Because it has all the UNIX networking, that is the file that had a completely separate game for DOS and Mac. I don't need anything that drastic, now. But there is a lot of very UNIX-specific stuff. One of those things is UNIX signal handling:

```
Error (active) E0020 identifier "SIGPIPE" is undefined rom.exe - x64-Debug
C:\cygwin64\home\bfelg\rom\src\comm.c 237
```

Here's the offending code in `comm.c`, near the top of `game_loop()`:

```c
    signal(SIGPIPE, SIG_IGN);
```

This is a fairly standard bit of UNIX network code that tells the the operating system, "Don't interrupt me to tell me about dead sockets or EOF's." `signal()` takes a "signal type" (category of interrupts), in this case `SIGPIPE` (file and/or socket descriptor events), and assigns it a "signal handler", in this case `SIG_IGN` (ignores that category of interrupts).

In short, Windows doesn't work this way. I will simply guard the code:

```c
#ifndef _MSC_VER
    signal(SIGPIPE, SIG_IGN);
#endif
```

### Undefined `socklen_t`

Next up is this error in `init_descriptor()`:

```
Error (active) E0020 identifier "socklen_t" is undefined rom.exe - x64-Debug
C:\cygwin64\home\bfelg\rom\src\comm.c 411
```

Here's the code:

```c
    socklen_t size;

    size = sizeof(sock);
    getsockname(control, (struct sockaddr*)&sock, &size);
```

This gives me a pretty good idea of what I need to do. `socklen_t` is an abstract type that is _often_ (but not _always_) `int`.

Here's the Win32 definition of `getsockname()`:

```c
int getsockname(
  [in]      SOCKET   s,
  [out]     sockaddr *name,
  [in, out] int      *namelen
);
```

The right way to fix this is to bring the disparate platforms into harmony with a macro that defines the type safely for both platforms:

```c
#ifdef _MSC_VER 
#include <winsock.h>
#define SOCKLEN int
#else
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>
#define SOCKLEN socklen_t
#endif
```

### Interlude &mdash; Disable Spurious Deprecation Warnings

Right now, I have a flood of these:

```
Warning	C4996 'sprintf': This function or variable may be unsafe. Consider using sprintf_s instead. To disable deprecation, use _CRT_SECURE_NO_WARNINGS. See online help for details.
```

Microsoft deviates frequently from POSIX, and often in the name of security. In this case, however, they want me to use a functions like `sprintf_s()` instead of `sprintf()`, or `fopen_s()` instead of `fopen()`.

These are "secure" alternatives, and have been made a part of the C Standard (since C11) as an optional extension. However, Microsoft's own implementation is at odds with the Standard, itself, and to dubious benefit. Therefore, other compilers have refused to implement these functions, and there is little point in trying to support two ways of doing _every_ POSIX function.

> Also, the POSIX functions aren't "deprecated"; the "secure" variants are merely an alternative (albeit only supported on one platform).

Therefore, I turn off the warning, and go about my day:

```cmake
if (MSVC)
    add_compile_options(/W3 /std:c17 )
    add_definitions(-D_CRT_SECURE_NO_WARNINGS -D NOCRYPT)
```

### Win32 Programming Administrivia

There are a lot of errors that come from WinSock and the headers it calls. 

I'll add more stuff to the MSVC block in `comm.c`:

```c
#ifdef _MSC_VER 
#undef UNICODE
#define WIN32_LEAN_AND_MEAN
#pragma comment(lib,"ws2_32.lib")
#include <windows.h>
#include <winsock.h>
#include <stdint.h>
#include <io.h>
#define SOCKLEN int
#else
#include <fcntl.h>
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>
#define SOCKLEN socklen_t
#endif
```

Let's take the new lines, each in turn:

```c
#undef UNICODE
```

This disables Unicode and forces all characters and string functions to assume 8-bit `char`. By default, all Win32 API calls use Unicode, and that would have created a headache for me with constant back and forth between `char` and `_TCHAR` (aka `wchar_t`).

> And because I don't want to debug with `s0t0r0i0n0g0s0 0t0h0a0t0 0l0o0o0k0 0l0i0k0e0 0t0h0i0s0.0`

```c
#define WIN32_LEAN_AND_MEAN
```

This is a bit of boilerplate for Win32 applications that instructs Win32 libraries to prevent header files from bringing in stuff that is not _always_ needed, like RPC, OLE, WinSpool, etc. It also excludes networking, but I can override that by explicitly bringing the relevant headers in, myself (as I do here).

Basically, anytime you are including `windows.h`, you _probably_ want this line _before_ you include it.

```c
#pragma comment(lib,"ws2_32.lib")
```

The pragma tells the compiler to dynamically link against the native WinSock2 library. This is needed to avoid "undefined reference" errors in the linker.

I'm using mostly legacy WinSock functions because they look like POSIX functions (and are often drop-in replacements). Nevertheless, everything in the background is really WinSock2. That means we need to link against that library.

Why link here and now in CMake? I could, probably _should_; but this has been the platform convention for some time.

```c
#include <windows.h>
#include <winsock.h>
```

I omitted it earlier, but `windows.h` needs to be included before `winsock.h`. I hate it when order matters, but it does on MSVC, and often.

Finally, I added `stdint.h` and `io.h` (that last one is a non-ANSI, MSVC-specific file) to do stuff later for error handling.

> Left as an exercise to the reader: These macros sit right above the `!defined(MSVC)` guard around `crypt.h`. It's pretty trivial to absorb that `#if` into this `#else`.

### Non-Blocking IO Control

The next error I get is this:

```
Error (active) E0020 identifier "O_NDELAY" is undefined rom.exe - x64-Debug	C:\cygwin64\home\bfelg\rom\src\comm.c 431
Error (active) E0020 identifier "F_SETFL" is undefined rom.exe - x64-Debug	C:\cygwin64\home\bfelg\rom\src\comm.c 431
```

Here's the code in `init_descriptor()`:

```c
#if !defined(FNDELAY)
#define FNDELAY O_NDELAY
#endif

    if (fcntl(desc, F_SETFL, FNDELAY) == -1) {
        perror("New_descriptor: fcntl: FNDELAY");
        return;
    }
```

This command tells a POSIX system how to handle file descriptors. Specifically, it sets a "no delay" flag on the master file descriptor when in non-blocking IO mode (I won't get into that here, but I intend to do a write-up on blocking vs non-blocking socket programming).

When `FNDELAY` (aka `O_NDELAY`) is set, read operations on non-blocking descriptors with no data in the buffer return 0 immediately. Normally, they would treat this as an error situation, setting `errno` and returning `-1`. But since we _expect_ this to happen, this is very much undesirable behavior.

However, because 0 is interpreted as an EOF, this can cause a spurious closed socket signal to be raised.... that is, unless you disable signal pipes (as, you'll recall, ROM _does_). 

Now, is this actually needed? The way ROM works; it doesn't attempt to read from a file descriptor unless a poll shows there is input on it. Therefore, it knows that a `0` is always an actual EOF. That means that if ROM used `O_NONBLOCK `, the modern, POSIX-compliant equivalent to `O_NDELAY`, we would never see a `-1` from a read operation. That, in turn, means the original signal pipe handling code is unnecessary (this is a conjecture; I haven't tried it yet).

In any event, this code is useless to MSVC. I will wrap it in a `#ifndef _MSC_VER` guard.

### Interfering with Library Headers

I like to put external API headers (`#include <filename>`) below project headers (`#include <filename>`) because I often want (or need) to set or unset macros that will be used by those headers (case in point: `UNICODE` and `WIN32_LEAN_AND_MEAN`).

However, it caused a pickle here, and it was not easy to track down:

```
Error C2143 syntax error: missing ':' before 'constant'	C:\cygwin64\home\bfelg\rom\src\out\build\x64-Debug\src C:\Program Files (x86)\Windows Kits\10\include\10.0.22000.0\um\winnt.h 6494	
```
 An error in a `winnt.h`? Where???

 ```c
 typedef union _ARM64_NT_NEON128 {
    struct {
        ULONGLONG Low;
        LONGLONG High;
    } DUMMYSTRUCTNAME;
    double D[2];
    float  S[4];
    WORD   H[8];
    BYTE   B[16];
} ARM64_NT_NEON128, *PARM64_NT_NEON128;
 ```

 I puzzled, and puzzled, until my puzzler was sore. I rearranged includes. I added some. I removed some. But _nothing_ made this error go away.

 And then I remembered the _flags_.

 See, in ROM, everything has a bunch of attributes (some have _dozens_ of them). A player character or a monster can be stunned, prone, sleeping, _and_ on fire, all at the same time. But since `bool` is secretly an `int`, its too costly to store them that way.

 So ROM uses an `int` for each category of flags and treats it as a bit-vector, with each bit representing whether the flag is set or not. Very standard, very simple C.

 The problem is what ROM does to "help" implementors make new flags.

 This is example is from `merc.h`:

```c
 /*
 * Extra flags.
 * Used in #OBJECTS.
 */
#define ITEM_GLOW               (A)
#define ITEM_HUM                (B)
#define ITEM_DARK               (C)
#define ITEM_LOCK               (D)
#define ITEM_EVIL               (E)
#define ITEM_INVIS              (F)
#define ITEM_MAGIC              (G)
#define ITEM_NODROP             (H)
#define ITEM_BLESS              (I)
#define ITEM_ANTI_GOOD          (J)
#define ITEM_ANTI_EVIL          (K)
```

wut...

```c
/* RT ASCII conversions -- used so we can have letters in this file */

#define A                       1
#define B                       2
#define C                       4
#define D                       8
#define E                       16
#define F                       32
#define G                       64
#define H                       128

#define I                       256
#define J                       512
#define K                       1024
#define L                       2048
#define M                       4096
#define N                       8192
#define O                       16384
#define P                       32768

#define Q                       65536
#define R                       131072
#define S                       262144
#define T                       524288
#define U                       1048576
#define V                       2097152
#define W                       4194304
#define X                       8388608

#define Y                       16777216
#define Z                       33554432
#define aa                      67108864 /* doubled due to conflicts */
#define bb                      134217728
#define cc                      268435456
#define dd                      536870912
#define ee                      1073741824
```

Oh, dear. Now the broken lines in `winnt.h` makes sense:

```c
    double D[2];
    float  S[4];
    WORD   H[8];
    BYTE   B[16];
```

This will not do. And now that I recall, we had a big problem back in the day when we overflowed the number of available bits in an `int`.

I want to delete all of those macros and replace them with this:

```c
#define BIT(x) (1 << x)
```

> If you are afraid of `BIT` also polluting header files, consider `FLAG_BIT` or `ROM_BIT`, etc. But then consider whether the same problem could happen with `SET_BIT` and `REMOVE_BIT`.
>
> If ever there was a C++-ism I could port to C, it would be namespaces.

But replacing hundreds and hundreds of references is dangerous; if you right-click on, say, `A` and do "Find All References", you'll see it spiders out into system headers. That won't do. Likewise, a global Search/Replace will also grab letters in strings (even with "Match Whole Word" enabled).

So I have to do it the hard way: I delete each flag, one by one, changing broken references them according to this chart:

| Old | New | Value |
| :--- | :--- | :--- |
| `A`  | `BIT(0)`  | `1` |
| `B`  | `BIT(1)`  | `2` |
| `C`  | `BIT(2)`  | `4` |
| `D`  | `BIT(3)`  | `8` |
| `E`  | `BIT(4)`  | `16` |
| `F`  | `BIT(5)`  | `32` |
| `G`  | `BIT(6)`  | `64` |
| `H`  | `BIT(7)`  | `128` |
| `I`  | `BIT(8)`  | `256` |
| `J`  | `BIT(9)`  | `512` |
| `K`  | `BIT(10)` | `1024` |
| `L`  | `BIT(11)` | `2048` |
| `M`  | `BIT(12)` | `4096` |
| `N`  | `BIT(13)` | `8192` |
| `O`  | `BIT(14)` | `16384` |
| `P`  | `BIT(15)` | `32768` |
| `Q`  | `BIT(16)` | `65536` |
| `R`  | `BIT(17)` | `131072` |
| `S`  | `BIT(18)` | `262144` |
| `T`  | `BIT(19)` | `524288` |
| `U`  | `BIT(20)` | `1048576` |
| `V`  | `BIT(21)` | `2097152` |
| `W`  | `BIT(22)` | `4194304` |
| `X`  | `BIT(23)` | `8388608` |
| `Y`  | `BIT(24)` | `16777216` |
| `Z`  | `BIT(25)` | `33554432` |
| `aa` | `BIT(26)` | `67108864` |
| `bb` | `BIT(27)` | `134217728` |
| `cc` | `BIT(28)` | `268435456` |
| `dd` | `BIT(29)` | `536870912` |
| `ee` | `BIT(30)` | `1073741824` |

Note that there is no flag for the 32nd bit (`BIT(31)`)

The easiest way to fix them is to navigate to error provoked by them, and do a Find/Replace _in file_ (Ctrl + H). They tend to be clustered in only 3 files: `merc.h`, `tables.c`, and `const.c`. Care must be taken that occurrences in string literals are not altered. A global Find/Replace will ruin your dev environment, and your day.

Once everything is done, I only have four remaining errors.

### Dealing with `void*`

The four errors are actually one error, in two places, reported two different ways:

```
Error E0852 expression must be a pointer to a complete object type 
C:\cygwin64\home\bfelg\rom\src\db.c 2345
Error C2036 'void *': unknown size
C:\cygwin64\home\bfelg\rom\src\db.c 2345
```

Here's the code in question, in `alloc_mem()` in `db.c`:

```c
pMem += sizeof(*magic);
```

`pMem` is defined in scope like so:

```c
void* pMem;
int* magic;
```

This is an interesting one. Here's the whole function, for context:

```c
/*
 * Allocate some ordinary memory,
 *   with the expectation of freeing it someday.
 */
void* alloc_mem(int sMem)
{
    void* pMem;
    int* magic;
    int iList;

    sMem += sizeof(*magic);

    for (iList = 0; iList < MAX_MEM_LIST; iList++) {
        if (sMem <= rgSizeList[iList]) break;
    }

    if (iList == MAX_MEM_LIST) {
        bug("Alloc_mem: size %d too large.", sMem);
        exit(1);
    }

    if (rgFreeList[iList] == NULL) { pMem = alloc_perm(rgSizeList[iList]); }
    else {
        pMem = rgFreeList[iList];
        rgFreeList[iList] = *((void**)rgFreeList[iList]);
    }

    magic = (int*)pMem;
    *magic = MAGIC_NUM;
    pMem += sizeof(*magic);

    return pMem;
}
```

Memory in ROM is handled by recycle pools, each one being a linked list of freely available memory. Each pool, or "bucket", has an assigned size. Each element in the bucket is the same size.

This function works like so:
1. `sMem` (the desired size of allocated memory) is increased by the size of a pointer. That's a "tax" on each allocated object, as the memory manager uses the first few bytes to store a pointer.
2. It loops through the bucket sizes to find which is the smallest bucket that can fulfill the desired memory allocation.
3. If no buckets can fulfill the request, it bails.
4. If the selected bucket has no "free" items, it calls `perm_malloc()`, which creates memory that cannot be freed, but it is _very_ fast (I have often used this trick).
5. Otherwise, if it _does_ find a free entry in that bucket, it grabs it and stores its address in `pMem`. 
6. The prefix of the item it grabbed from the bucket is a pointer to the next item in the bucket. It overwrites the bucket head (heh, bucket-head), with this value. Thus, `rgFreeList[bucket_size]` now points to the _next_ item in the bucket. `pMem` is now the owner of that memory.
7. It overwrites the "'next' pointer" at the beginning of pMem with `MAGIC_NUM` (`52571214`).

> What makes this number (`52571214`) "magic" for memory allocation is that neither it, nor any number derived from a subset of its bytes, can be misinterpreted as a valid pointer address:
>
> | x / 16 | Quotient | Remainder | Digit # |
> | :--- | :--- | :--- | :--- |
> |(52571214)/16| 3285700  | 14	| 0 |
> |(3285700)/16	| 205356   | 4	| 1 |
> |(205356)/16	| 12834	   | 12	| 2 |
> |(12834)/16	| 802	   | 2	| 3 |
> |(802)/16	    | 50	   | 2	| 4 |
> |(50)/16	    | 3	       | 2	| 5 |
> |(3)/16	    | 0	       | 3  | 6 |
>
> Basically, we can always inspect the prefix to see if it's allocated memory that needs to go back into the bucket, because there can never be allocated memory starting at that address.
>
> It's actually ingenious, and it's _fast_.

So what's the issue with the code above? For some time now, the C Standards have frowned on using `void*` for... pretty much anything. Actually, they don't think much of _pointers_, either. That's why C11 added a new type for handing pointer addresses: `uintptr_t`. In fact, this type is tailor-made for what this function is trying to do.

Here's the fix:

```c
(uintptr_t)pMem += sizeof(*magic);
```

`uintptr_t` is that I use to make clear my intention to use pointer arithmatic.

I make a similar change in `free_mem()`:

```c
(uintptr_t)pMem -= sizeof(*magic);
```

### Unresolved Reference: `gettimeofday()`

Here's one that I choose to "phone it in":

```
Error LNK2019 unresolved external symbol gettimeofday referenced in function game_loop
C:\cygwin64\home\bfelg\rom\src\out\build\x64-Debug\comm.c.obj 1
```

I'm not going to lie; I shamelessly [pulled this code from StackOverflow](https://stackoverflow.com/a/26085827).

```c
#ifdef _MSC_VER
////////////////////////////////////////////////////////////////////////////////
// This implementation taken from StackOverflow user Michaelangel007's example:
//    https://stackoverflow.com/a/26085827
//    "Here is a free implementation:"
////////////////////////////////////////////////////////////////////////////////
static int gettimeofday(struct timeval* tp, struct timezone* tzp)
{
    // Note: some broken versions only have 8 trailing zero's, the correct epoch
    // has 9 trailing zero's. This magic number is the number of 100 nanosecond 
    // intervals since January 1, 1601 (UTC) until 00:00:00 January 1, 1970.
    static const uint64_t EPOCH = ((uint64_t)116444736000000000ULL);

    SYSTEMTIME  system_time;
    FILETIME    file_time;
    uint64_t    time;

    GetSystemTime(&system_time);
    SystemTimeToFileTime(&system_time, &file_time);
    time = ((uint64_t)file_time.dwLowDateTime);
    time += ((uint64_t)file_time.dwHighDateTime) << 32;

    tp->tv_sec = (long)((time - EPOCH) / 10000000L);
    tp->tv_usec = (long)(system_time.wMilliseconds * 1000);
    return 0;
}
////////////////////////////////////////////////////////////////////////////////
#endif
```

This is pretty much _the_ way to do this correctly, and I cannot imagine a homegrown solution would be any better.

### Unresolved `random()` and `srandom()`:

These methods were problem children for me on Linux, so I'm not surprised I'm still having issues with them:

```
Error LNK2019 unresolved external symbol srandom referenced in function init_mm
C:\cygwin64\home\bfelg\rom\src\out\build\x64-Debug\db.c.obj 1
```

Under Windows, to use a pseudo-random number generator (PRNG), you have to pull in a Crypto library. Like I said, I am not ready to do that yet. Besides, crypto algorithms are not necessary for me in a _dice roller_. I'm not looking for cryptographical security in a dice-roller; I'm looking for _speed_. 

In a previous post I said I didn't want to use the old-and-busted `rand()`/`srand()` in place of `random()`/`srandom()`, but the truth is that `random()` and `srandom()` are _also_ busted; just not as much.

So, maybe it's time I look for a new (and cross-platform) solution. 

Now, I do not have the intellectual capability of _writing_ a PRNG; it's a field mostly driven by white-papers written by PhD's, and is, for the most part, inaccessible to the layman. Also, it's just not an area of interest for me.

In the end, I have settled on what could be considered a _contraversial_ choice: [PCG Random](https://www.pcg-random.org/). The website (and the author) make _bold_ claims about its performance, and critics say the comparisons made to other PRNG's are cherry-picked. However, independent analysis shows that PCG is really quite excellent on all areas. And, in the end, it's widely accepted by _programmers_ if not the ivory tower of academia, and that's good enough for me.

Therefore, I choose to remove `random()` and `srandom()` and switch to PCG for all platforms. 

[Here is the GitHub for the minimal C implementation](https://github.com/imneme/pcg-c-basic). I only require `pcg_basic.h` and `pcg_basic.c` (and I need to add `pcg_basic.c` to `CMakeLists.txt`).

I add this to `db.c`:

```c
#include "pcg_basic.h"
```

And I replace `init_mm()` and `number_mm()` like so:

```c
void init_mm()
{
    int rounds = 5;

    // Seed with external entropy -- the time and some program addresses
    pcg32_srandom(time(NULL) ^ (intptr_t)&printf, (intptr_t)&rounds);
}

long number_mm(void)
{
    return pcg32_random();
}
```

And with that, all errors are taken care of. Granted, there are still 164 warnings, 35 messages, and everything that was hidden by `-D_CRT_SECURE_NO_WARNINGS`, but this has been _huge_.

### One Last Thing: `\dev\null`

Remember this in `merc.h`?

```c
 // TODO: Research how to reserve a stream in MSVC without /dev/null.
#define NULL_FILE       "/dev/null"     // To reserve one stream
```

Now is the time to take care of it:

```c
#ifndef _MSC_VER
    #define NULL_FILE   "/dev/null"     // To reserve one stream
#else
    #define NULL_FILE   "nul"
#endif
```

That's basically the Windows equivalent of `/dev/null`.

### So, How did I Do?

Now that I have have ROM building with no errors (and have ignorantly and happily hidden all warnings and messages), it's time for me to take it out for a spin:

```
PS C:\cygwin64\home\bfelg\rom\area> ..\src\out\build\x64-Debug\rom.exe
Init_socket: socket: No error
```

_/sad-trombone.wav_

Just because it built without errors doesn't mean it's right. Also, I did state at the beginning that I would have to rewrite significant portions of the network code.

So, let's get to it.

## Adding WinSock

I've already added all the `#include`s needed. So here we go:

### Adding the `SOCKET` Abstract Type

Win32 uses `SOCKET` (`UINT_PTR`) to hold a file descriptor, whereas UNIX uses `int`. Intend to unify the code as much as possible, so as to reduce in-code macros.

I add just below the UNIX-specific `#include`s in `comm.c`:

```c
#define SOCKET int
```

I then change the following function signatures (both definitions _and_ forward declarations):

```c
void game_loop(SOCKET control);
void init_descriptor(SOCKET control);
SOCKET init_socket(int port);
write_to_descriptor(SOCKET desc, char* txt, int length);
```

This is an unfortunate addition I need in `merc.h`, due to `description_data` living there:

```c
#ifdef _MSC_VER
#include <winsock.h>
#else
#define SOCKET int
#endif
```

Later, in `description_data`, I change `descriptor`'s type:

```c
    SOCKET descriptor;
```

### Handling WinSock Errors

WinSock has its own error handling system, which I abstract into `GetLastWinSockError()` which I put near the top of `comm.c`:

```c
#ifdef _MSC_VER
static void PrintLastWinSockError()
{
    char msgbuf[256] = "";
    int err = WSAGetLastError();

    FormatMessage(FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS, // flags
        NULL,           // lpsource
        err,            // message id
        MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), // languageid
        msgbuf,         // output buffer
        sizeof(msgbuf), // size of msgbuf, bytes
        NULL);          // va_list of arguments

    if (!*msgbuf)
        sprintf(msgbuf, "%d", err);

    printf("Error: %s", msgbuf);
}
#endif
```

I will be calling this many times during implementation.

### Retrofitting `main()`

Most of my work for adding WinSock support will be in `comm.c`.

The first thing I need to do, before I make _any_ WinSock calls, is initializing the API. I do it before anything else, near the top of `main()` just after I declared `control`:

```c
    SOCKET control;

#ifdef _MSC_VER
    WORD wVersionRequested;
    WSADATA wsaData;
    int err;

    /* Use the MAKEWORD(lowbyte, highbyte) macro declared in Windef.h */
    wVersionRequested = MAKEWORD(2, 2);

    err = WSAStartup(wVersionRequested, &wsaData);
    if (err != 0) {
        printf("WSAStartup failed with error: %d\n", err);
        exit(1);
    }
#endif
```

When everything is done, WinSock needs to be cleaned up. I put this near the end of `main()`, just after we close the `control` socket:

```c
    close(control);

#ifdef _MSC_VER
    WSACleanup();
#endif
```

### Closing Sockets

In the code above, something is wrong. `close()` is the POSIX way; it doesn't work in WinSock. There I need ot use `closesocket()`.

To fix this issue, I add another macro called `CLOSE_SOCKET` split between the platforms:

```c
#ifdef _MSC_VER
#pragma comment(lib, "Ws2_32.lib")
#include <windows.h>
#include <winsock.h>
#include <stdint.h>
#include <io.h>
#define CLOSE_SOCKET closesocket
#define SOCKLEN int
#else
#include <netdb.h>
#include <netinet/in.h>
#include <sys/socket.h>
#include <sys/time.h>
#include <unistd.h>
#define CLOSE_SOCKET close
#define SOCKLEN socklen_t
#define SOCKET int
#endif
```

Then I change the `close()` call at the end of `main()`:

```c
CLOSE_SOCKET(control);
```

### Creating the Listener Socket

Moving on to `init_socket()`,  I alter `fd`'d declaration:

```c
    SOCKET fd;
    int errno;
```

After every `perror()`, I grab the last WinSock error:

```c
#ifdef _MSC_VER
        PrintLastWinSockError();
#endif
```

I also change instances of `close()` to `CLOSE_SOCKET()`.

Similar changes are also made in `game_loop()`, changing `maxdesc` from `int` to `SOCKET`. This then, necessitates a cast to `int` where it needs to.

For instance, here's the `select()` call, with all the above changes added:

```c
        if (select((int)maxdesc + 1, &in_set, &out_set, &exc_set, &null_time) < 0) {
            perror("Game_loop: select: poll");
#ifdef _MSC_VER
            PrintLastWinSockError();
#endif
            CLOSE_SOCKET(control);
            exit(1);
        }
```

### Changes to Read/Write

First, in `read_from_descriptor()`, I add alternate WinSock logic:

```c
#ifdef _MSC_VER
        nRead = recv(d->descriptor, d->inbuf + iStart, 
            (int)(sizeof(d->inbuf) - 10 - iStart), 0);
#else
        nRead = read(d->descriptor, d->inbuf + iStart,
            sizeof(d->inbuf) - 10 - iStart);
#endif
```

And then do likewise in `write_to_descriptor()`:

```c
#ifdef _MSC_VER
        if ((nWrite = send(desc, txt + iStart, nBlock, 0)) < 0) {
            PrintLastWinSockError();
#else

        if ((nWrite = write(desc, txt + iStart, nBlock)) < 0) {
#endif
            perror("Write_to_descriptor");
            return false;
        }
```

## Other Last Minute Things

Here are a few little gotchas that bit me during this process that prevented ROM from running successfully in Windows. Here they are, in short bytes:

### Fixing "Hit Enter to Continue"

It didn't do anything until I changed two instances of...

```c
if (d->showstr_point)
```

...to...

```c
if (d->showstr_point && *d->showstr_point != '\0')
```

Both of these were in `comm.c`.

### Fixing Timer Synchronization

In `comm.c`, near the end of `game_loop()` I needed to add separate logic for MSVC:

```c
    if (secDelta > 0 || (secDelta == 0 && usecDelta > 0)) {
#ifdef _MSC_VER
        long int mSeconds = (secDelta * 1000) + (usecDelta / 1000);
        if (mSeconds > 0) {
            Sleep(mSeconds);
        }
#else
        struct timeval stall_time;

        stall_time.tv_usec = usecDelta;
        stall_time.tv_sec = secDelta;
        if (select(0, NULL, NULL, NULL, &stall_time) < 0) {
            perror("Game_loop: select: stall");
            exit(1);
        }
#endif
    }
```

### The Player Folder

I had a nasty crash that took some sleuthing to get to the bottom of. As it turns out, I was missing the `rom/player` folder, which meant that attempts to save player files were crashing. The folder didn't exist of a rule in `.gitignore` to ignore that folder.

First, I fix `.gitignore` by removing this line:

```
player/*
```

Then I add the `player/` folder (in `rom/`) with a 0-byte `.dummy` that I will check into git. This will ensure the creation of the folder with a git pull.

### Connection Closes After Entering Password

This one was a doozy. After entering my new character's name, the connection closed. The server didn't crash, and allowed me to reconnect. No error was logged. What gives?

During debugging, I found that `read_from_buffer()` was failing the `ISASCII` check, which then threw out my input string. `read_from_buffer()` then interpreted the empty string as shenanigans and disconnected me.

Here's what I had as my `ISASCII` implementation in `strings.h`:

```c
#define ISASCII(c) ((unsigned char)(c) & ~0x7F)
```

MSVC did not like that. This is a more portable replacement that doesn't care what the type is:

```c
#define ISASCII(c) (c >= 0 && c <= 127)
```

And since I'm already in `strings.h`, I add `_MSC_VER` to the the `__CYGWIN__` guard:

```c
#if defined(__CYGWIN__) || defined(_MSC_VER)
```

If I don't do this, ROM will crash on `ISPRINT` when it gets the telent command `IAC WONT ECHO` along with the player's password from the client.

## Validating the Changes With GCC/Clang

I made some deep cuts in the code. I need to valdate my changes on the other platforms.

### Cygwin

Unfortunately, GCC on Cygwin doesn't like some of the changes I made in `db.c`:

```
/home/bfelg/rom/src/db.c: In function ‘alloc_mem’:
/home/bfelg/rom/src/db.c:2346:21: error: lvalue required as left operand of assignment
 2346 |     (uintptr_t)pMem += sizeof(*magic);
      |                     ^~
/home/bfelg/rom/src/db.c: In function ‘free_mem’:
/home/bfelg/rom/src/db.c:2360:21: error: lvalue required as left operand of assignment
 2360 |     (uintptr_t)pMem -= sizeof(*magic);
      |                     ^~
```

Huh. Casting the lval for unary addition made sense to me at the time, but now it seems odd. It must be an MSVC extension. I may have misled myself by looking at this in `free_mem()`:

```c
*((void**)pMem) = rgFreeList[iList];
```

However, that's a pure assignment operation, so it might work differently.

So I'll do this: in `alloc_mem()`, I trade out `pMem`:

```c
    //void* pMem;
    uintptr_t mem_addr;
```

_Now_ it's an appropriate lval. But now I need to add some rval casts:

```c
    if (rgFreeList[iList] == NULL) { 
        mem_addr = (uintptr_t)alloc_perm(rgSizeList[iList]); 
    }
    else {
        mem_addr = (uintptr_t)rgFreeList[iList];
        rgFreeList[iList] = *((void**)rgFreeList[iList]);
    }

    magic = (int*)mem_addr;
    *magic = MAGIC_NUM;
    mem_addr += sizeof(*magic);

    return (void*)mem_addr;
```

 `uintptr_t`, as a scalar integer value, is somewhat misnamed; it's an _address_, not a _pointer_. It has same bits but has different semantic meaning to the compiler. That's why I named the new value `mem_addr`.

I make similar changes to `free_mem()`; this time copying `pMem` to `mem_addr` and then back again after doing the pointer arithmetic:

```c
void free_mem(void* pMem, int sMem)
{
    int iList;
    int* magic;
    uintptr_t mem_addr = (uintptr_t)pMem;

    mem_addr -= sizeof(*magic);
    magic = (int*)mem_addr;

    if (*magic != MAGIC_NUM) {
        bug("Attempt to recyle invalid memory of size %d.", sMem);
        bug((char*)mem_addr + sizeof(*magic), 0);
        return;
    }

    *magic = 0;
    sMem += sizeof(*magic);

    for (iList = 0; iList < MAX_MEM_LIST; iList++) {
        if (sMem <= rgSizeList[iList]) break;
    }

    if (iList == MAX_MEM_LIST) {
        bug("Free_mem: size %d too large.", sMem);
        exit(1);
    }

    pMem = (void*)mem_addr;
    *((void**)pMem) = rgFreeList[iList];
    rgFreeList[iList] = pMem;

    return;
}
```

### GCC and Clang on Linux

On Linux, I run the following commands to check for warnings:

```
./config gcc && ./build clean
./config clang && ./build clean
```

There are no errors, and no warnings. 

## End Notes

Now I have ROM building (and running!) under MSVC on Windows. However, there are still hundreds of warning to take care of whe compiling under MSVC. That will be the next part of this series. Once all four platforms (GCC/Linux, Clang/Linux, Cygwin, and MSVC) are building without errors or warnings, it will be time to set up ROM for profiling, and then start hacking.

([Here is the code](https://github.com/bfelger/Mud98/tree/70717079d2b4bb3a1e3ab1fd692e165ee8cf6e07) with updates from this post.)

Next: [Part 6](pt-6-msvc-2)

Copyright 2023, Brandon Felger

