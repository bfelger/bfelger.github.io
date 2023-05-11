---
layout: default
title:  "Resurrecting ROM Pt. 4 - Cross-Platform Compilation With CMake"
date:   2023-05-11 20:00:00 -0500
categories: mud C CMake
---

# Resurrecting ROM Pt. 4 &mdash; Cross-Platform Compilation With CMake

_Soon, the last vestiges of TAB characters will be erased from this project forever._

This post picks up where [Part 3](pt-2-dusting-off-cobwebs) left off. [Here is the code](https://github.com/bfelger/rom/tree/addcc5c67e81940dff838f45e12fbae22c75ecff) that resulted from the work.

Makefiles have been working well for us, so far. But as I increase the number of build targets, so too will the number of Makefiles required. CMake is a cross-platform build solution that, unlike Makefile, has official support in Visual Studio. This will be crucial when I port to MSVC, as it avoids the creation of solution and project files.

I will have _one_ CMake file, covering _all_ platforms.

## Configuring CMake

First, I create a rudimentary `CMakeLists.txt` in the `src` folder:

```cmake
#
# ROM CMakeLists.txt
#
cmake_minimum_required (VERSION 3.10)

project(rom)

add_executable(rom act_comm.c act_enter.c act_info.c act_move.c act_obj.c 
    act_wiz.c alias.c ban.c comm.c const.c db.c db2.c effects.c fight.c flags.c
    handler.c interp.c lookup.c magic.c magic2.c music.c note.c recycle.c 
    save.c scan.c skills.c special.c tables.c update.c)
```

Since I'm working on a fresh Ubuntu image, I needed to install CMake and Ninja:

```bash
sudo apt install cmake
sudo apt install ninja-build
```

> Obviously this is Ubuntu-specific.

There are many "generators" for CMake; I chose Ninja because it's what Visual Studio uses with CMake, and I like to be consistent across the board.

Next I create new `.build` folder under `src`, then create the CMake files from `src`:

```bash
cmake -G "Ninja Multi-Config" -B .build
```

I get this output in response:

```
-- The C compiler identification is GNU 11.3.0
-- The CXX compiler identification is GNU 11.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/bfelger/rom/Rom24/src/.build
```

Now I build it like so, using the generated Ninja files in `.build`:

```bash
ninja -C .build
```

It builds _fast!!!_. Much faster than `make`.

But I have an error:

```
ninja: Entering directory `.build'
[30/30] Linking C executable Debug/rom
FAILED: Debug/rom
: && /usr/bin/cc -g  CMakeFiles/rom.dir/Debug/act_comm.c.o CMakeFiles/rom.dir/Debug/act_enter.c.o CMakeFiles/rom.dir/Debug/act_info.c.o CMakeFiles/rom.dir/Debug/act_move.c.o CMakeFiles/rom.dir/Debug/act_obj.c.o CMakeFiles/rom.dir/Debug/act_wiz.c.o CMakeFiles/rom.dir/Debug/alias.c.o CMakeFiles/rom.dir/Debug/ban.c.o CMakeFiles/rom.dir/Debug/comm.c.o CMakeFiles/rom.dir/Debug/const.c.o CMakeFiles/rom.dir/Debug/db.c.o CMakeFiles/rom.dir/Debug/db2.c.o CMakeFiles/rom.dir/Debug/effects.c.o CMakeFiles/rom.dir/Debug/fight.c.o CMakeFiles/rom.dir/Debug/flags.c.o CMakeFiles/rom.dir/Debug/handler.c.o CMakeFiles/rom.dir/Debug/interp.c.o CMakeFiles/rom.dir/Debug/lookup.c.o CMakeFiles/rom.dir/Debug/magic.c.o CMakeFiles/rom.dir/Debug/magic2.c.o CMakeFiles/rom.dir/Debug/music.c.o CMakeFiles/rom.dir/Debug/note.c.o CMakeFiles/rom.dir/Debug/recycle.c.o CMakeFiles/rom.dir/Debug/save.c.o CMakeFiles/rom.dir/Debug/scan.c.o CMakeFiles/rom.dir/Debug/skills.c.o CMakeFiles/rom.dir/Debug/special.c.o CMakeFiles/rom.dir/Debug/tables.c.o CMakeFiles/rom.dir/Debug/update.c.o -o Debug/rom   && :
/usr/bin/ld: CMakeFiles/rom.dir/Debug/act_info.c.o: in function `do_password':
/home/bfelger/rom/Rom24/src/act_info.c:2435: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/act_info.c:2450: undefined reference to `crypt'
/usr/bin/ld: CMakeFiles/rom.dir/Debug/comm.c.o: in function `nanny':
/home/bfelger/rom/Rom24/src/comm.c:1102: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/comm.c:1204: undefined reference to `crypt'
/usr/bin/ld: /home/bfelger/rom/Rom24/src/comm.c:1223: undefined reference to `crypt'
/usr/bin/ld: CMakeFiles/rom.dir/Debug/interp.c.o:(.data.rel.ro+0xb78): undefined reference to `do_heal'
collect2: error: ld returned 1 exit status
ninja: build stopped: subcommand failed.
```

Note that these are not parser errors, but linker errors. If you recall from [Part 1](pt-1-compile-gcc), I had to add `lcrypt` to the end of the linker line. But I don't have that here. How do I add library dependencies in CMake?

It's pretty easy; I add this line to `CMakeLists.txt`:

```cmake
target_link_libraries(rom crypt)
```

Note that I don't have to worry about order, at all. CMake and Ninja together figure it out for me.

The only remaining error is the missing `do_heal()` from `interp.o`.

Oops! I was missing `healer.c`. Here's the fixed list:

```cmake
add_executable(rom act_comm.c act_enter.c act_info.c act_move.c act_obj.c 
    act_wiz.c alias.c ban.c comm.c const.c db.c db2.c effects.c fight.c flags.c
    handler.c healer.c interp.c lookup.c magic.c magic2.c music.c note.c 
    recycle.c save.c scan.c skills.c special.c tables.c update.c)
```

Note that anytime I change `CMakeLists.txt`, I have to regenerate the Ninja build files:

```bash
cmake -G "Ninja Multi-Config" -B .build .
ninja -C .build
```

Which, theoretically, isn't something I have to do very often. But since I am in the "setup" phase, I will be doing it quite a bit.

And with that, I now have my `rom` binary in the `.build/Debug` folder.

### Alternate Configs

Normally, I would need to re-run CMake for each release type, but since I used Ninja Multi-Config, the targets are already ready to be built.

For instance, to build `Release`:

```bash
 cmake --build .build --config Release
```

In this case, I didn't need to call `ninja` directly, at all. The `Release` build is now in `.build/Release`.

Another nice thing about this system is that my `src` folder is unpolluted by binaries and object files. 

But I do need to remember to add the `.build` folder to `.gitignore`.

### Parity With Makefile

In Makefile with GCC and Clang, I was able to set extended warnings and whatnot. I can do the same thing with CMake. This snippet comes straight from the documentation:

```cmake
if (MSVC)
    # warning level 4 and all warnings as errors
    add_compile_options(/W4)
else()
    # lots of warnings and all warnings as errors
    add_compile_options(-Wall)
endif()
```

> Exercise to the reader: Add `-Wextra -pedantic -Werror` for GCC/Clang and `/Wall /WX` for MSVC to get even more nit-picky warnings from GCC and Clang, and for those warnings to be treated as errors. Some don't have elegant solutions, so be warned. Eventually I _will_ enable these flags, as they are pretty much standard.

Notice I have a little future-proofing with MSVC support. But there's one more flag I really want to add:

```cmake
if (MSVC)
    # warning level 4 and all warnings as errors and set C17
    add_compile_options(/W4 /std:c17)
else()
    # lots of warnings and all warnings as errors and set C23
    add_compile_options(-Wall -std=c2x)
endif()
```

I added a flag to set the highest C Standard that each set of compilers supports (MSVC lags behind quite a bit). Where this leads to an actual difference in code, I will need to use macros. However, I don't think this will be often.

> The flag is `c2x` because it might not become official in 2023; it might become 2024 or some such. It's a placeholder. MSVC Doesn't even fully support C99, so setting `c17` there is more like wish-casting.

## New GCC/Linux Warnings with C23

Bumping the C Standard to C23 had changed how the code is parsed and compiled. The default for GCC and Clang is C99; there have been quite a few iterations since then, and that means we have new errors and warnings.

### Undefined `isacii()`

Here's the first one:

```
/home/bfelger/rom/Rom24/src/comm.c: In function ‘read_from_buffer’:
/home/bfelger/rom/Rom24/src/comm.c:624:18: warning: implicit declaration of function ‘isascii’ [-Wimplicit-function-declaration]
  624 |         else if (isascii(d->inbuf[i]) && ISPRINT(d->inbuf[i]))
      |                  ^~~~~~~
```

`isascii()` tests an integer so see if it's a 7-bit `char`. Basically, this will only fail if I am testing sign-bit-enabled telnet character. Interesting that previous compilations under C99 with `-Wall` didn't catch this.

To fix this, first I correct an omission I made when I first made `strings.h`:

```c
#include <ctype.h>
```

All of the `isxxxxx()` methods live in `ctype.h`, and I believe that each file should include its own dependencies (and if you don't want it to spider out and grab the world... well... be careful).

I then add the following to the `__CYGWIN__` and non-`__CIGWIN__` blocks, respectively:

```c
#ifdef __CYGWIN__
    #define ISALPHA(c) isalpha((unsigned char)c)
```

```c
#else
    #define ISASCII(c) isascii(c)
```

Then I update the call in `read_from_buffer()`:

```c
        else if (ISASCII(d->inbuf[i]) && ISPRINT(d->inbuf[i]))
```

### Clean Builds in CMake

With `make`, I had a way of seeing warnings from source files that completed without errors by using the `clean` option I added to `Makefile`:

```bash
make clean && make
```

CMake gives me a similar facility right out of the box by adding `--clean-first` to the build step:

```bash
 cmake --build .build --clean-first
```

This cleans out the binaries and forces a compilation from scratch.

And now, back to fixing errors and warnings.

### Undefined `isascii()`

And now, a totally new, totally differe&mdash;

```
In file included from /home/bfelger/rom/Rom24/src/comm.c:49:
/home/bfelger/rom/Rom24/src/comm.c: In function ‘read_from_buffer’:
/home/bfelger/rom/Rom24/src/strings.h:25:24: warning: implicit declaration of function ‘isascii’ [-Wimplicit-function-declaration]
   25 |     #define ISASCII(c) isascii(c)
      |                        ^~~~~~~
/home/bfelger/rom/Rom24/src/comm.c:624:18: note: in expansion of macro ‘ISASCII’
  624 |         else if (ISASCII(d->inbuf[i]) && ISPRINT(d->inbuf[i]))
      |                  ^~~~~~~
```

Oh.

I didn't fix it.

But I did all the things! I included `ctype.h`, I added a cast to play nice with signed `char`; what more does it want?!?

Taking a look as [the specification for `isascii()`](https://pubs.opengroup.org/onlinepubs/9699919799/functions/toascii.html) I notice a little tag marked "OB XSI". That's defined thusly under POSIX:

> [OB] Obsolescent<br>
> The functionality described may be removed in a future version of this volume of POSIX.1-2017. Strictly Conforming POSIX Applications and Strictly Conforming XSI Applications shall not use obsolescent features.

Ew. So, this was obsolesced (and removed from GCC) because `isascii()` returns non-locale-safe characters, and POSIX is all about fully locale-aware code. That is, if your app can't handle arbitrary locales, then it is not POSIX-compliant. Thus, `isascii()`'s primary use (enforcing "POSIX" Locale [aka C Locale]) circumvents the intention of POSIX.

But that's too bad; as I mentioned before, telnet (and other RFC-compliant modes of communication, like HTTP, POP, IMAP, and SMTP) can _only_ use those bottom 7 bits.

So, I am adding this to `strings.h`, after the `#ifdef __CYGWIN__`...`#else` blocks:

```c
// isascii() is marked OB by POSIX, but we still need to test for telnet-
// friendly characters.
#define ISASCII(c) ((unsigned char)(c) & ~0x7F)
```

I also remove the two previous definitions of  `ISASCII()`. That takes care of that warning once and for all.

### It's My Old Friend `crypt()`

This thing is a like a bad rash; it just keeps coming back:

```
/home/bfelger/rom/Rom24/src/comm.c: In function ‘nanny’:
/home/bfelger/rom/Rom24/src/comm.c:1102:20: warning: implicit declaration of function ‘crypt’ [-Wimplicit-function-declaration]
 1102 |         if (strcmp(crypt(argument, ch->pcdata->pwd), ch->pcdata->pwd)) {
      |                    ^~~~~
/home/bfelger/rom/Rom24/src/comm.c:1102:20: warning: passing argument 1 of ‘strcmp’ makes pointer from integer without a cast [-Wint-conversion]
 1102 |         if (strcmp(crypt(argument, ch->pcdata->pwd), ch->pcdata->pwd)) {
      |                    ^~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
      |                    |
      |                    int
```

Somehow I lost my definition of `crypt()` (and so help me, POSIX, if I find out you had anything to do with it...).

And it is, _sort of_. `libcrypt` is in a set of "optional" libraries that POSIX implementations may (or may not) have to include. GNU chose to replace `libcrypt` with `libxcrypt` as a drop-in replacement. BUT, if it doesn't exist, I can include `crypt.h` to get the `libcrypt` definitions based on the absence of `_XOPEN_CRYPT`.

So one way to do it is to modify the `__CYGWIN__` test for `crypt.h` in both `comm.c` and `ac_info.c`:

```c
#ifndef _XOPEN_CRYPT
#include <crypt.h>
#endif
```

**Note that `#ifdef` is now `#ifndef`; I want to check if `_XOPEN_CRYPT` is NOT defined!**

I'm sure I'll have to deal with this, again. But for now, I'm building without errors, again.

> There are ways of hacking `CMakeLists.txt` to try `libxcrypt` and fall back to `libcrypt` if need be. I'm holding off on that, for now.

### Fixing `rand()` and `srand()`

The final warnings have to do with the PRNG:

```
/home/bfelger/rom/Rom24/src/db.c: In function ‘init_mm’:
/home/bfelger/rom/Rom24/src/db.c:2718:5: warning: implicit declaration of function ‘srandom’; did you mean ‘srand’? [-Wimplicit-function-declaration]
 2718 |     srandom(time(NULL) ^ getpid());
      |     ^~~~~~~
      |     srand
/home/bfelger/rom/Rom24/src/db.c: In function ‘number_mm’:
/home/bfelger/rom/Rom24/src/db.c:2724:12: warning: implicit declaration of function ‘random’; did you mean ‘rand’? [-Wimplicit-function-declaration]
 2724 |     return random() >> 6;
      |            ^~~~~~
      |            rand
```

It asks if I meant to say `srand()` and `rand()`, respectively, and I _do not_. This is another area where POSIX marks `srandom()` and `random()` as "XSI", which means they are optional extensions. When I set the `-std` flag on GCC, it got _very_ strict on what it will allow. To use a newer C Standard _and_ use optional POSIX extensions, I need to pass a different, GCC-specific set of compile flags.

Now I need to break up Clang and GCC by changing `CMakeLists.txt` like so:

```cmake
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_compile_options(/W4 /WX /std:c17)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-Wall -std=gnu2x -D_GNU_SOURCE)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-Wall -std=c2x)
endif()
```

And with that, ROM now builds in CMake for Linux under GCC using the C23 draft standard (or as much as GNU has implemented so far in GCC 11.3).

## Building With CMake Using Clang

To configure Ninja to produce Clang builds, I need to reconfigure Ninja. But to do that effectively, I have to delete the CMake cache, first:

```bash
rm .build/* -rf
```

And then I call the CMake configuration again, but this time explicitly setting Clang as my toolchain:

```bash
 cmake -G "Ninja Multi-Config" -B .build . -DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang
```

In response, I get this:

```
-- The C compiler identification is Clang 14.0.0
-- The CXX compiler identification is Clang 14.0.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/clang - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/clang++ - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/bfelger/rom/Rom24/src/.build
```

> Why set the C++ compiler? ROM doesn't use any C++, and I think rushing to convert C projects to C++ throws a lot of baby out with the bath water. Nevertheless, I set it for two reasons:
> - Consistency, because I have OCD tendencies
> - I want to keep my options open in case I ever change my mind on keeping it pure C.

Now I can build like normal:

```bash
cmake --build .build --clean-first
```

And, wow. Only two warnings. Clang was much happier with those POSIX extensions than GCC was. 

### Missing `srandom()` Again

Here's the warnings I got:

```
[21/31] Building C object CMakeFiles/rom.dir/Debug/db.c.o
/home/bfelger/rom/Rom24/src/db.c:2718:5: warning: implicit declaration of function 'srandom' is invalid in C99 [-Wimplicit-function-declaration]
    srandom(time(NULL) ^ getpid());
    ^
/home/bfelger/rom/Rom24/src/db.c:2724:12: warning: implicit declaration of function 'random' is invalid in C99 [-Wimplicit-function-declaration]
    return random() >> 6;
           ^
2 warnings generated.
```

Now, that can't be a coincidence.

So, as it turns out, while Clang can use LLVM's own custom C++ STL, it has no C library of its own. That is, it shares whatever C library GCC is using. In this case, `glibc`. That also means they share headers and _preprocessor flags_. Now, the implementation of the C Standard is not part of the C header library, but of the compiler itself.

> `glibc` supports BSD and SysV, both of which are POSIX implementations that _require_ `srandom()` and `random()` implementations. So before I set the higher C standard, the compiler was lax about letting BSD stuff "leak" through. Afterward, it tightened it down. 

So, my first attempt to fix is the keep Clang's `-std` flag the same, but port over the `_GNU_SOURCE` macro that enables those `glibc` extensions:

```cmake
if (CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
    add_compile_options(/W4 /WX /std:c17)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-Wall -std=gnu2x -D_GNU_SOURCE)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    add_compile_options(-Wall -std=c2x -D_GNU_SOURCE)
endif()
```

And that works.

### What Were Those Commands, Again?

I kind of miss the days of `make clean && make` for it's simplicity. The flags for CMake and Ninja aren't immediately intuitive. And while I know that, in time, I will learn them by heart, _I don't want to._

I will make two new files. The First one is a text file called `config`:

```bash
#!/bin/bash

compiler=""

for arg in "$@" 
do
    if [[ ${arg,,} == "gcc" ]]
    then
        compiler="-DCMAKE_CXX_COMPILER=g++ -DCMAKE_C_COMPILER=gcc"
    elif [[ ${arg,,} == "clang" ]]
    then
        compiler="-DCMAKE_CXX_COMPILER=clang++ -DCMAKE_C_COMPILER=clang"
    else
        echo "Unknown argument '${arg,,}'."
        exit
    fi

done

mkdir -p .build
rm .build/* -rf
cmake -G "Ninja Multi-Config" -B .build . $compiler
```

I run this to make it an executable script:

```bash
chmod +x config
```

Now I can run the following commands:

```bash
./config        # Default CMake/Ninja config (GCC)
./config gcc
./config clang
```

Each option clears the CMake cache before reconfiguring Ninja.

The next file is called `build`:

```bash
#!/bin/bash

clean_opt=""
build_cfg="--config Debug"

for ARG in "$@" 
do
    if [[ ${ARG,,} == "clean" ]]
    then
        clean_opt="--clean-first"
    elif [[ ${ARG,,} == "release" ]] || [[ ${ARG,,} == "rel" ]]
    then
        build_cfg="--config Release"
    elif [[ ${ARG,,} == "debug" ]] || [[ ${ARG,,} == "deb" ]]
    then
        build_cfg="--config Debug"
    elif [[ ${ARG,,} == "relwithdebinfo" ]] || [[ ${ARG,,} == "relwithdeb" ]] || [[ ${ARG,,} == "reldeb" ]]
    then
        build_cfg="--config RelWithDebInfo"
    else
        echo "Unknown argument '${ARG,,}'."
        exit
    fi

done

cmake --build .build $clean_opt $build_cfg
```

And make it executable:

```bash
chmod +x build
```

Now I can build using any of these settings (this is not an exhaustive list):

```bash
./build                 # --config Debug
./build debug           # --config Debug
./build clean           # --clean-first --config Debug
./build clean release   # --clean-first --config Release
./build rel             # --config Release
./build reldeb          # --config RelWithDebInfo
```

That's much easier. And while I still have my good ol' Makefiles, they aren't going to see much use from me.

## Building With CMake Using Cygwin

First, I have to run the Cygwin install and select the `cmake` and `ninja` packages.

> Good sign: when editing source files to run on Cygwin, I typically use Visual Studio 2022 as my IDE. As soon as I opened the folder, it picked up my `CMakeLists.txt` and started configuring the project for it. I'm looking forward to not having to configure much, thanks to CMake/Ninja.

The first thing to try, I suppose, is our plain-Jane `./config`:

```bash
$ ./config
./config: line 2: $'\r': command not found
./config: line 4: $'\r': command not found
./config: line 6: syntax error near unexpected token `$'do\r''
'/config: line 6: `do
```

Ew. Carraige returns!

For both `config` and `build`, I open the file in Notepad++ and select `Edit` | `EOL Conversion` | `Unix (LF)`, then save and close.

But this next part is important to keep it from happening again:

```bash
git config --global core.autocrlf false
```

Now; let's try this again:

```bash
$ ./config
-- The C compiler identification is GNU 11.3.0
-- The CXX compiler identification is GNU 11.3.0
-- Detecting C compiler ABI info
-- Detecting C compiler ABI info - done
-- Check for working C compiler: /usr/bin/cc - skipped
-- Detecting C compile features
-- Detecting C compile features - done
-- Detecting CXX compiler ABI info
-- Detecting CXX compiler ABI info - done
-- Check for working CXX compiler: /usr/bin/c++.exe - skipped
-- Detecting CXX compile features
-- Detecting CXX compile features - done
-- Configuring done
-- Generating done
-- Build files have been written to: /home/bfelg/rom/src/.build
```

So far, so good.

```bash
$ ./build
[31/31] Linking C executable Debug/rom.exe
```

Awesome. Zero extra work, and I have full cross-compatability with Windows using Cygwin.

Now I'm ready to tackle the Herculean task of adding full MSVC support, and only two series parts behind.

([Here is the code](https://github.com/bfelger/rom/tree/f03fa77b6e4779dc2e03b0b88c93bbe7d2cc0c3b) with updates from this post.)

Copyright 2023, Brandon Felger
