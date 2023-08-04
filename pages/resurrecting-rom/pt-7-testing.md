---
layout: default
title:  "Resurrecting ROM Pt. 7 - Benchmarks and Unit Tests"
date:   2023-05-22 20:00:00 -0500
categories: mud C
---

# Resurrecting ROM Pt. 7 &mdash; Benchmarks and Unit Tests

This post picks up where [Part 6](pt-6-msvc-2) left off. [Here is the code](https://github.com/bfelger/rom/tree/ac968f668dd3f5c96eee55d6e855641d6a8ba496) that resulted from that work.

Now that I have ROM up and running, with no warnings or errors, I want to start noodling around in the code. Improving legacy code requires two things that, at the moment, ROM is not set up to perform: benchmarks and unit tests.

Both of these are special operations in ROM that alter how in runs, so I want add options to run these tests without interfering with the normal operation of the application.

The easiest way to to that is to add command-line handling.

## Adding Command-Line Handling

Right now, ROM only takes one (optional) command-line argument:

```c
    /*
     * Get the port number.
     */
    port = 4000;
    if (argc > 1) {
        if (!is_number(argv[1])) {
            fprintf(stderr, "Usage: %s [port #]\n", argv[0]);
            exit(1);
        }
        else if ((port = atoi(argv[1])) <= 1024) {
            fprintf(stderr, "Port number must be above 1024.\n");
            exit(1);
        }
    }
```

I want to extend this logic by adding the ability to select to explicitly set the port as a named command-line parameter like so:

```c
   /*
     * Get the command line arguments.
     */
    port = 4000;
    char* port_str = NULL;

    if (argc > 1) {
        for (int i = 1; i < argc; i++) {
            if (is_number(argv[i])) {
                port = atoi(argv[i]);
            }
            else if (!strcmp(argv[i], "-p")) {
                if (++i < argc) {
                    port_str = argv[i];
                }
            }
            else if (!strncmp(argv[i], "--port=", 7)) {
                port_str = argv[i] + 7;
            }
        }
    }

    if (port_str) {
        if (is_number(port_str)) {
            port = atoi(port_str);
        }
        else {
            fprintf(stderr, "Must specify a number for port.\n");
            exit(1);
        }
    }

    if (port <= 1024) {
        fprintf(stderr, "Port number must be above 1024.\n");
        exit(1);
    }
```

Here, I keep the old default argument logic, but also add the ability to set the port via the command line arguments `-p 4000` or `--port=4000`.

But it would be nice to have a more generic, extensible way of handling CLI arguments.

### Arbitrary Working Directory

Something that annoys me, currently, is the requirement that ROM has to be executed from the `area` directory. This is made all the harder by the fact that the `rom` executable no longer lives in `src`, its traditional home. One of the ways this impacts me is that when I debug ROM in MSVC, I have to start it, then attach to the process manually. That's quite annoying.

I can fix this by adding a new command-line parameter telling ROM where the area files are. Then I can run from an arbitrary location. For instance, running from MSVC's `src\out\Debug` directory:

```
rom.exe -a ../../../../area/
```

To do this, I need to add support for the `-a` command-line argument. First, I add this to the global variables in `comm.c`:

```c
char area_dir[256] = DEFAULT_AREA_DIR;
```

That's an arbitrary number of characters. It would be anything, I suppose. Windows has a hard limit of 256 characters for a filename + path, but there could also be back-pathing (like with `../`), so more might be necessary.

I define `DEFAULT_AREA_DIR` in `merc.h`, along with an extern of `area_dir`, as all my file saves will now refer to it:

```c
extern char area_dir[];

#define DEFAULT_AREA_DIR "./"
```

This default assumes that ROM is being run the way it is expected to, now: from the `area` directory.

I add two new options to the command-line handler, as well as the top-level variable to hold the string argument:

```c
    char* area_dir_str = NULL;
```

```c
            else if (!strcmp(argv[i], "-a")) {
                if (++i < argc) {
                    area_dir_str = argv[i];
                }
            }
            else if (!strncmp(argv[i], "--area-dir=", 7)) {
                area_dir_str = argv[i] + 11;
            }
```

I use an intermediate value so I can change it in its own buffer if it's missing a trailing slash (`/`):

```c
    if (area_dir_str) {
        size_t len = strlen(area_dir_str);
        if (area_dir_str[len - 1] != '/' && area_dir_str[len - 1] != '\\')
            sprintf(area_dir, "%s/", area_dir_str);
        else
            sprintf(area_dir, "%s", area_dir_str);
    }
```

Now I need to retrofit all references to files in the program (other than `NULL_FILE`) to include the new `area_dir` as a prefix (the other folders, such as `player`, are prefixed with `../` so they will work out-of-the-box). 

The trivial case is like this line from `comm.c`:

```c
sprintf(strsave, "%s%s", PLAYER_DIR, capitalize(ch->name));
```

I simply add `area_dir` as a prefix:

```c
sprintf(strsave, "%s%s%s", area_dir, PLAYER_DIR, capitalize(ch->name));
```

This one, from `ban.c`, is a little trickier:

```c
if ((fp = fopen(BAN_FILE, "w")) == NULL) { 
    perror(BAN_FILE); 
}
```

Because `BAN_FILE` is a literal, but `area_dir` is not, concatenation of the two cannot be done at compile-time. I have to create a buffer and combine them, there:

```c
char ban_file[256];
sprintf(ban_file, "%s%s", area_dir, BAN_FILE);
if ((fp = fopen(ban_file, "w")) == NULL) { 
    perror(ban_file); 
}
```

Note that I have to replace every usage of `BAN_FILE` in this context with `ban_file`, including the `unlink()` at the end of the function:

```c
if (!found) 
    unlink(ban_file);
```

Here's an even more convoluted example from `db.c`:

```c
    /*
     * Read in all the area files.
     */
    {
        FILE* fpList;

        if ((fpList = fopen(AREA_LIST, "r")) == NULL) {
            perror(AREA_LIST);
            exit(1);
        }

        for (;;) {
            strcpy(strArea, fread_word(fpList));
            if (strArea[0] == '$') break;

            if (strArea[0] == '-') { fpArea = stdin; }
            else {
                if ((fpArea = fopen(strArea, "r")) == NULL) {
                    perror(strArea);
                    exit(1);
                }
            }
```

In this case, I need two buffers, one for `area.lst` and one to use for each individual area listed inside it:

```c
    /*
     * Read in all the area files.
     */
    {
        FILE* fpList;
        char area_list[256];
        char area_file[256];
        sprintf(area_list, "%s%s", area_dir, AREA_LIST);
        if ((fpList = fopen(area_list, "r")) == NULL) {
            perror(area_list);
            exit(1);
        }

        for (;;) {
            strcpy(strArea, fread_word(fpList));
            if (strArea[0] == '$') break;

            if (strArea[0] == '-') {
                fpArea = stdin; 
            }
            else {
                sprintf(area_file, "%s%s", area_dir, strArea);
                if ((fpArea = fopen(area_file, "r")) == NULL) {
                    perror(area_file);
                    exit(1);
                }
            }
```

This change is an area where testing is non-optional; a single missed file will crash the entire program.

Right now is when I wish I had comprehensive unit tests to catch these problems. I'll get to that later.

## Unit-Test and Benchmarks

Now that I have command-line parameters available, I'd like to add an option to run unit tests, either as part of the normal boot process of ROM, or as a stand-alone feature.

To accommodate this, I add two new globals in `comm.c`:

```c
bool rt_opt_benchmark = false;
bool rt_opt_noloop = false;
```

To run benchmarks in stand-alone mode, I can set both the these values to `true`.

Next I add the actual command-line arguments:

```c
            else if (!strcmp(argv[i], "--benchmark")) {
                rt_opt_benchmark = true;
            }
            else if (!strcmp(argv[i], "--benchmark-only")) {
                rt_opt_benchmark = true;
                rt_opt_noloop = true;
            }
            else if (argv[i][0] == '-') {
                char* opt = argv[i] + 1;
                while (*opt != '\0') {
                    switch (*opt) {
                    case 'b':
                        rt_opt_benchmark = true;
                        break;
                    case 'B':
                        rt_opt_benchmark = true;
                        rt_opt_noloop = true;
                        break;
                    default:
                        fprintf(stderr, "Unknown option '-%c'.\n", *opt);
                        exit(1);
                    }
                    opt++;
                }
            }
            else {
                fprintf(stderr, "Unknown argument '%s'.\n", argv[i]);
                exit(1);
            }
```

Note that I added a new option for combined single-letter arguments (like `tar -xzvf` or some such). In this case, `-b` and `-B` are equivalent to `--benchmark` and `--benchmark-only`, respectively. Right now it might seem silly to have a loop to iterate through these characters if I only have one possible option, but my time will be saved down the road if I ever add new options.

I think `rt_opt_noloop` is self-explanatory: it do not run the game-loop at all, or accept socket connections. But that means there is a mess of code in `comm.c` that I do not need if I run with `--benchmark-only`. Problem: a lot of that code happens before execution even gets to the argument handler. It needs to be moved.

Here is the chunk that needs to be moved:

```c
#ifdef _MSC_VER
    WORD wVersionRequested;
    WSADATA wsaData;
    int err;

    /* Use the MAKEWORD(lowbyte, highbyte) macro declared in Windef.h */
    wVersionRequested = MAKEWORD(2, 2);

    err = WSAStartup(wVersionRequested, &wsaData);
    if (err != 0) {
        /* Tell the user that we could not find a usable */
        /* Winsock DLL.                                  */
        printf("WSAStartup failed with error: %d\n", err);
        exit(1);
    }
#endif

    /*
     * Memory debugging if needed.
     */
#if defined(MALLOC_DEBUG)
    malloc_debug(2);
#endif

    /*
     * Init time.
     */
    gettimeofday(&now_time, NULL);
    current_time = (time_t)now_time.tv_sec;
    strcpy(str_boot_time, ctime(&current_time));

    /*
     * Reserve one channel for our use.
     */
    if ((fpReserve = fopen(NULL_FILE, "r")) == NULL) {
#ifdef _MSC_VER
        printf("Please create a file named '%s' in the executing directory.", NULL_FILE);
#endif
        perror(NULL_FILE);
        exit(1);
    }
```

I move it to immediately after the command-line argument handler, but I guard the WinSock initialization against `rt_opt_no_loop`:

```c
#ifdef _MSC_VER
    if (!rt_opt_noloop) {
        WORD wVersionRequested;
        WSADATA wsaData;
        int err;

        /* Use the MAKEWORD(lowbyte, highbyte) macro declared in Windef.h */
        wVersionRequested = MAKEWORD(2, 2);

        err = WSAStartup(wVersionRequested, &wsaData);
        if (err != 0) {
            /* Tell the user that we could not find a usable */
            /* Winsock DLL.                                  */
            printf("WSAStartup failed with error: %d\n", err);
            exit(1);
        }
    }
#endif
```

I do the same thing with the main game loop:

```c

    if (!rt_opt_noloop) {
        control = init_socket(port);
        sprintf(log_buf, "ROM is ready to rock on port %d.", port);
        log_string(log_buf);
        game_loop(control);
        CLOSE_SOCKET(control);
    }
```

Note that I moved the `init_socket()` call from earlier in the code.

I don't have any immediate needs for benchmarks, but I want to create a simple one to use as a reference implementation.

### Adding Benchmarks

I start by added a new file, `benchmark.h`:

```c
////////////////////////////////////////////////////////////////////////////////
// benchmark.h
//
// Utilities for gathering metrics to benchmark code changes
////////////////////////////////////////////////////////////////////////////////

#pragma once
#ifndef ROM__BENCHMARK_H
#define ROM__BENCHMARK_H

#include <stdbool.h>
#include <time.h>

typedef struct {
    struct timespec start;
    struct timespec stop;
    bool running;
} Timer;

struct timespec elapsed(Timer* timer);
void start_timer(Timer* timer);
void stop_timer(Timer* timer);

#endif // !ROM__BENCHMARK_H
```

It defines a simple timer that I can use the benchmark how long certain processes take. The idea is that I can compare "before" and "after" numbers to see if the future changes I implement make a difference or not.

Next I create `benchmark.c` and get it primed:

```c
////////////////////////////////////////////////////////////////////////////////
// benchmark.c
//
// Utilities for gathering metrics to benchmark code changes
////////////////////////////////////////////////////////////////////////////////

#include "benchmark.h"
```

The first function starts the timer:

```c
void start_timer(Timer* timer)
{
    if (!timer->running) {
        clock_gettime(CLOCK_MONOTONIC, &(timer->start));
        timer->running = true;
    }
}
```

> NOTE: The code repo for this commit has a bug in `start_timer()`: the conditional in this function is lacking the NOT (`!`) operator. This is fixed in Latest.

The second function stops the timer:

```c
void stop_timer(Timer* timer)
{
    if (timer->running) {
        clock_gettime(CLOCK_MONOTONIC, &(timer->stop));
        timer->running = false;
    }
}
```

The last function calculates the difference between them:

```c
struct timespec elapsed(Timer* timer)
{
    struct timespec temp = { 0 };

    if (!timer->running)
        return temp;

    if ((timer->stop.tv_nsec - timer->start.tv_nsec) < 0) {
        temp.tv_sec = timer->stop.tv_sec - timer->start.tv_sec - 1;
        temp.tv_nsec = 1000000000 + timer->stop.tv_nsec - timer->start.tv_nsec;
    }
    else {
        temp.tv_sec = timer->stop.tv_sec - timer->start.tv_sec;
        temp.tv_nsec = timer->stop.tv_nsec - timer->start.tv_nsec;
    }
    return temp;
}
```

I could have commented this better, but the first conditional tests to see if subtracting nanoseconds results in a negative value. If it does, I subtract a single second from `tv_sec` and add that many nanoseconds to `tv_nsec`.

Now, MSVC has a problem: it does not have the POSIX function `clock_gettime()`. I once again cheat and pull in code from the internet. I place this above the other functions in `benchmark.c`:

```c
#ifdef _MSC_VER
#include <windows.h>
#include <winsock.h>
#define CLOCK_MONOTONIC		        1
#define CLOCK_PROCESS_CPUTIME_ID    2

#define MS_PER_SEC      1000ULL     // MS = milliseconds
#define US_PER_MS       1000ULL     // US = microseconds
#define HNS_PER_US      10ULL       // HNS = hundred-nanoseconds (e.g., 1 hns = 100 ns)
#define NS_PER_US       1000ULL

#define HNS_PER_SEC     (MS_PER_SEC * US_PER_MS * HNS_PER_US)
#define NS_PER_HNS      (100ULL)    // NS = nanoseconds
#define NS_PER_SEC      (MS_PER_SEC * US_PER_MS * NS_PER_US)

////////////////////////////////////////////////////////////////////////////////
// This implementation taken from StackOverflow user jws's example:
//    https://stackoverflow.com/a/51974214
// I only implemented the CLOCK_MONOTONIC version.
////////////////////////////////////////////////////////////////////////////////
static int clock_gettime(int X, struct timespec* tv)
{
    static LARGE_INTEGER ticksPerSec;
    LARGE_INTEGER ticks;

    if (!ticksPerSec.QuadPart) {
        QueryPerformanceFrequency(&ticksPerSec);
        if (!ticksPerSec.QuadPart) {
            errno = ENOTSUP;
            return -1;
        }
    }

    QueryPerformanceCounter(&ticks);

    tv->tv_sec = (long)(ticks.QuadPart / ticksPerSec.QuadPart);
    tv->tv_nsec = (long)(((ticks.QuadPart % ticksPerSec.QuadPart) * NS_PER_SEC) 
        / ticksPerSec.QuadPart);

    return 0;
}
////////////////////////////////////////////////////////////////////////////////
#endif
```

Now I need to hook up the function calls to make it work. I include the new header in `comm.c`:

```c
#include "benchmark.h"
```

Timing the boot sequence is pretty trivial, at this point; I just need to book-end the call to `boot_db()` in `main()`:

```c
    Timer boot_timer = { 0 };
    if (rt_opt_benchmark)
        start_timer(&boot_timer);

    boot_db();

    if (rt_opt_benchmark) {
        stop_timer(&boot_timer);
        struct timespec timer_res = elapsed(&boot_timer);
        sprintf(log_buf, "Boot time: "TIME_FMT"s, %ldns.", 
            timer_res.tv_sec, timer_res.tv_nsec);
        log_string(log_buf);
    }
```

And with that, I am now set up move into actual feature adds for ROM.

([Here is the code](https://github.com/bfelger/rom/tree/98569541809c13d9f6f7c9b755541a033e4fb7f1) with updates from this post. Note that it has the aforementioned error that is fixed in Latest.)

Next: [Part 8](pt-8-mt-sockets)

Copyright 2023, Brandon Felger