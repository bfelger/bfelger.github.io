---
layout: default
title:  "Resurrecting ROM Pt. 8 - Preparing for Secure Sockets"
date:   2023-08-01 20:00:00 -0500
categories: mud C
---

# Resurrecting ROM Pt. 8 &mdash; Preparing for Secure Sockets

This post picks up where [Part 7](pt-7-testing) left off. [Here is the code](https://github.com/bfelger/rom/tree/98569541809c13d9f6f7c9b755541a033e4fb7f1) that resulted from that work.

Before I do anything further with ROM, I need to fix a glaring problem that plagued me back in _Ye Olden Tymes_: clear-text telnet sockets that broadcast my flippin' _password_ to the whole world. Now, there are a number of ways to skin this cat, and it seems that a popular option is SSH/SSL tunneling.  I prefer a more built-in approach so ROM can run securely out-of-the-box.

I evaluated a number of options. SSH would have been nice, but LibSSH is poorly documented, and LibSSH2 only supports client sockets. Therefore, I am focusing my efforts on TLS connections with OpenSSL.

## A Brief Primer on Encrypted Connections

Currently, ROM implements "raw" sockets in non-blocking mode. This is necessary because ROM is single-threaded. When `accept()` is called in blocking mode, it sits and waits for a new connection. Traditional POSIX apps fork themselves after accepting a connection, spawning a new process that opens a new listener while the original does work with its single client. This is the simplest form of multi-user service and the the easiest to implement. But a MUD requires interconnected communication between clients, and doing this over separate processes is difficult. This complicates the net code, as we are required to run in a non-blocking mode.

In non-blocking mode, open sockets need to be polled for incoming connections. If there are no new client connections, ROM skips calling `accept()` for that tick. Incoming communications from existing client connections are also polled, as ROM doesn't call `read()` (or `recv()` in MSVC) unless there are communications to read (as these are also normally blocking functions).

This works because `accept()` is very fast in raw sockets. It can be done along with other telnet I/O serially in the same process.

But encrypted communications throw a monkey-wrench into this: new connections are required to perform a lengthy "handshake", where the server and the client exchange keys and validate them against stored certificates. Usually this is automated; but depending on the client, there may be a prompt to the user asking the accept a key from the new source. There is no guarantee that the handshake will be done in a timely manner.

What I need is the ability to handle this handshake in an asynchronous manner. But before I take care of that, I want to do some code re-organization.

## Separating the Network Code

Right now, the core game loop is tightly coupled with the network code. Implementing TLS will make this enough of a bugbear of `#ifdef`'s, already; I don't want that problem to leak into the code game code.

First thing's first: I want move core game logic out of `comm.c` and into a new file called `main.c`, the way it should have been to start with.

> Remember to add all new code files to `CMakeLists.txt`.

Unlike previous new code files that I have added, this is not novel code, and I need to keep all the copyright information in the comments at the start of the file. I won't show that here, though, because it's boilerplate.

To start with, I move most of the include files from `comm.c`, and then some of the global variables and function definitions:

```c
// Global variables.
DESCRIPTOR_DATA* descriptor_list;   // All open descriptors
DESCRIPTOR_DATA* d_next;            // Next descriptor in loop
FILE* fpReserve;                    // Reserved file handle
bool god;                           // All new chars are gods!
bool merc_down = false;             // Shutdown
bool wizlock;                       // Game is wizlocked
bool newlock;                       // Game is newlocked
char str_boot_time[MAX_INPUT_LENGTH];
time_t current_time;                // time of this pulse

bool rt_opt_benchmark = false;
bool rt_opt_noloop = false;
char area_dir[256] = DEFAULT_AREA_DIR;

void game_loop(SockServer* server);

#ifdef _MSC_VER
int gettimeofday(struct timeval* tp, struct timezone* tzp);
#endif
```

Some of these, however, are still needed in `comm.c`:

```c
extern bool merc_down;                      // Shutdown
extern bool wizlock;                        // Game is wizlocked
extern bool newlock;                        // Game is newlocked
```

The `extern` keyword is basically a promise to the compiler that these items will be resolved by the linker with another translation unit (fancy word for "all the code in a file with all includes and text replacements"). That means that there is only one actual copy: the one in `main.c`.

At this point, I copy over `main()` from `main.c`, only this time I put it right under my global definitions, before any other methods. That is the way is _should_ be.

Then I also copy over `gettimeofday()` and `game_loop()`. In the process, I remove the `static` modifier on `gettimeofday()`, as it needs to be shared across TU's (translation units; get used to it :-P).

Because these functions reference a lot of pieces from `comm.c`, I need to make a `comm.h` that I can include from `main.c`:

```c
////////////////////////////////////////////////////////////////////////////////
// comm.h
////////////////////////////////////////////////////////////////////////////////

#pragma once
#ifndef ROM__COMM_H
#define ROM__COMM_H

#include "merc.h"

#ifdef _MSC_VER
#include <winsock.h>
#else
#include <sys/select.h>
#define SOCKET int
#endif

// Descriptor (channel) structure.
typedef struct descriptor_data {
    struct descriptor_data* next;
    struct descriptor_data* snoop_by;
    CHAR_DATA* character;
    CHAR_DATA* original;
    bool valid;
    char* host;
    SOCKET descriptor;
    int16_t connected;
    bool fcommand;
    char inbuf[4 * MAX_INPUT_LENGTH];
    char incomm[MAX_INPUT_LENGTH];
    char inlast[MAX_INPUT_LENGTH];
    int repeat;
    char* outbuf;
    size_t outsize;
    ptrdiff_t outtop;
    char* showstr_head;
    char* showstr_point;
} DESCRIPTOR_DATA;

void init_descriptor(SOCKET control);
SOCKET init_socket(int port);
void nanny(DESCRIPTOR_DATA* d, char* argument);
bool process_output(DESCRIPTOR_DATA* d, bool fPrompt);
void read_from_buffer(DESCRIPTOR_DATA* d);
bool read_from_descriptor(DESCRIPTOR_DATA* d);
void stop_idling(CHAR_DATA* ch);
bool write_to_descriptor(DESCRIPTOR_DATA* d, char* txt, size_t length);

#ifdef _MSC_VER
void PrintLastWinSockError();
#endif

#endif // ROM__COMM_H
```

I can now move these definitions out of `comm.c`.

One significant new change I slipped in here is `struct descriptor_data` (and its attendant alias `DESCRIPTOR_DATA`). I've stood on my soapbox often enough about `merc.h` that this sort of change was inevitable. Now that `comm.c` is focused solely on telnet comms, it becomes a more natural place to keep `DESCRIPTOR_DATA`. I remove its definition from `mer.c`, but leave its `typedef` in place. However, any TU that tries to access the _guts_ of `DESCRIPTOR_DATA` will now need to include `comm.h`. There are a lot of them.

I have now separated the core game logic from `comm.c`, but have not separated the socket code from `main.c`. I need to continue refactoring the code so that `main.c` has no awareness of the underlying socket implementation. Along the way, I'm going to forecast my needs with SSL and set myself up for future success.

To do this, I need to remove any reference to `SOCKET`, `fd_set`, or the socket/WinSock functions.

In `comm.h`, I add three new aggregate data-types:

```c
typedef struct sock_server_t {
    SOCKET control;
} SockServer;

typedef struct sock_client_t {
    SOCKET fd;
} SockClient;

typedef struct poll_data_t {
    fd_set in_set;
    fd_set out_set;
    fd_set exc_set;
    SOCKET maxdesc;
} PollData;
```

If you've inspected the socket code at all, it should be fairly obvious what these are, and how they are used. The names, to keep things simple, are unchanged from their current usage in `game_loop()` (with the exception of `fd`, which is referred to in `DESCRIPTOR_DATA` as `descriptor`. I find `descriptor.descriptor` to be incredibly unhelpful, and always have).

> Heretofore, I haven't trucked much with the names of existing types in ROM, mostly because it's confusing. Because these are new types of my own design, I'm freer to pursue my own philosophy:
> - `struct` names, if not anonymous, are `snake_case` suffixed with `_t`.
> - All `struct`s are defined with an inline `typedef` declaring a `CamelCase` alias.
> - Aliases are used exclusively because I hate prepending `struct` to everything.
>
> These are _my_ rules, not _your_ rules. The world of C is filled with opinionated grognards like me; you should decide for yourself what kind of opinionated grognard _you_ want to be.

I replace this in `DESCRIPTOR_DATA`:

```c
SOCKET descriptor
```

with this:

```c
SockClient client;
```

This is going to pay off when I go to implement TLS: there is more information what needs to go with the socket, but nothing outside the net code needs to know about it.

As part of this change, any reference to `d->descriptor` needs to be replaced with `d->client.fd`.

I also replace the definition of `control` in `main()`:

```c
SockServer server;
```

Any where that referred to `control` needs to be changed to `server.control`; but that doesn't make me very happy, as it still represents a tight coupling of the net code with the game logic. From now on, I want `main()` and `game_loop()` to only deal with `server`, and never its constituent components (whatever they may end up being).

### Repurposing `init_socket()` As `init_server()`

In `comm.c`, `init_socket()` sets up the main listener socket and binds it to the server. I'm going to consolidate a lot of initial net code into it. First, I rename it and change its signature, in both code file and header:

```c
void init_server(SockServer* server, int port)
```

I move the following code out of `main()` and into `init_server()`, just after local declarations:

```c
#ifndef _MSC_VER
    signal(SIGPIPE, SIG_IGN);
#else
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
```

Since this function returns `void` instead of `int`, I replace the final `return` statement:

```c
server->control = fd;
```

The former call in `main.c` is also replaced; I change this:

```c
control = init_socket(port);
```

...to this:

```c
init_server(&server, port);
```

### New Function: `close_server()`

I add a another function to `comm.c` that moves code out of the end of `main()`:

```c
void close_server(SockServer* server)
{
    CLOSE_SOCKET(server->control);

#ifdef _MSC_VER
    WSACleanup();
#endif
}
```

I also replace this code in `main.c` with this:

```c
close_server(&server);
```

### Updating `init_descriptor()`

Because I don't want function signatures to use `SockServer` instead of `SOCKET`, I'm changing `init_descriptor()`:

```c
void init_descriptor(SockServer* server)
```

Since `server` provides `control`, I can remove this line from `init_descriptor()`:

```c
SOCKET fd;
```

Instead of dealing with `fd`, I work directly on `server->control`.

### New Function: `can_write()`

I add a new function to `comm.c` and prototype it in `comm.h`:

```c
bool can_write(DESCRIPTOR_DATA* d, PollData* poll_data)
{
    return FD_ISSET(d->client.fd, &poll_data->out_set);
}
```

You may remember that `PollResult` is a new structure added to `comm.h` to handle values that were once local to `game_loop()`. This function checks to see if a client socket is available to have bytes sent to it.

### New Function: `has_new_conn()`

This function in `comm.c` checks to see if we can accept a new connection on the master socket:

```c
bool has_new_conn(SockServer* server, PollData* poll_data)
{
    return FD_ISSET(server->control, &poll_data->in_set);
}
```

Like the other new functions I'm adding here, this is all code that previously existed in `game_loop()`.

### New Function: `poll_server()`

Here is another big chunk of code that I'm removing from `game_loop()` and moving back into `comm.c`:

```c
void poll_server(SockServer* server, PollData* poll_data)
{
    static struct timeval null_time = { 0 };

    // Poll all active descriptors.
    FD_ZERO(&poll_data->in_set);
    FD_ZERO(&poll_data->out_set);
    FD_ZERO(&poll_data->exc_set);
    FD_SET(server->control, &poll_data->in_set);
    poll_data->maxdesc = server->control;
    for (DESCRIPTOR_DATA* d = descriptor_list; d; d = d->next) {
        poll_data->maxdesc = UMAX(poll_data->maxdesc, d->client.fd);
        FD_SET(d->client.fd, &poll_data->in_set);
        FD_SET(d->client.fd, &poll_data->out_set);
        FD_SET(d->client.fd, &poll_data->exc_set);
    }

    if (select((int)poll_data->maxdesc + 1, &poll_data->in_set, 
            &poll_data->out_set, &poll_data->exc_set, &null_time) < 0) {
        perror("poll_server");
#ifdef _MSC_VER
        PrintLastWinSockError();
#endif
        close_server(server);
        exit(1);
    }
}
```

Thanks to `PollData`, I can pass these `fd_set`s as an aggregate. This then enables me to chop up `game_loop()`s net code into discrete chunks and move them back out to `comm.c`. That's the work I'm going to do, now.

### Rename `process_output()` to `process_descriptor_output()`

I rename this function (and that's all there is to it) because I'm also adding another "process output" function, and I want to disambiguate them:

```c
bool process_descriptor_output(DESCRIPTOR_DATA* d, bool fPrompt)
```

### New Function: `process_client_output()`

This new function takes more code out of `game_loop()`:

```c
void process_client_output(PollData* poll_data)
{
    for (DESCRIPTOR_DATA* d = descriptor_list; d != NULL; d = d_next) {
        d_next = d->next;

        if ((d->fcommand || d->outtop > 0)
            && can_write(d, poll_data)) {
            if (!process_descriptor_output(d, true)) {
                if (d->character != NULL && d->connected == CON_PLAYING)
                    save_char_obj(d->character);
                d->outtop = 0;
                close_socket(d);
            }
        }
    }
}
```

### New Function: `process_client_input()`

This is the largest of the new functions, and culls the last of the net code from `game_loop()`:

```c
void process_client_input(SockServer* server, PollData* poll_data)
{
    // Kick out the freaky folks.
    for (DESCRIPTOR_DATA* d = descriptor_list; d != NULL; d = d_next) {
        d_next = d->next;
        if (FD_ISSET(d->client.fd, &poll_data->exc_set)) {
            FD_CLR(d->client.fd, &poll_data->in_set);
            FD_CLR(d->client.fd, &poll_data->out_set);
            if (d->character && d->connected == CON_PLAYING)
                save_char_obj(d->character);
            d->outtop = 0;
            close_socket(d);
        }
    }

    // Process input.
    for (DESCRIPTOR_DATA* d = descriptor_list; d != NULL; d = d_next) {
        d_next = d->next;
        d->fcommand = false;

        if (FD_ISSET(d->client.fd, &poll_data->in_set)) {
            if (d->character != NULL) d->character->timer = 0;
            if (!read_from_descriptor(d)) {
                FD_CLR(d->client.fd, &poll_data->out_set);
                if (d->character != NULL && d->connected == CON_PLAYING)
                    save_char_obj(d->character);
                d->outtop = 0;
                close_socket(d);
                continue;
            }
        }

        if (d->character != NULL && d->character->daze > 0)
            --d->character->daze;

        if (d->character != NULL && d->character->wait > 0) {
            --d->character->wait;
            continue;
        }

        read_from_buffer(d);
        if (d->incomm[0] != '\0') {
            d->fcommand = true;
            stop_idling(d->character);

            if (d->showstr_point && *d->showstr_point != '\0')
                show_string(d, d->incomm);
            else if (d->connected == CON_PLAYING)
                substitute_alias(d, d->incomm);
            else
                nanny(d, d->incomm);

            d->incomm[0] = '\0';
        }
    }
}
```

### The Svelte New `game_loop()`

First, I change `game_loop()`s signature:

```c
void game_loop(SockServer* server)
```

In `main()`, I change the function call to pass `server` instead of (the now-defunct) `control`. I change this:

```c
 if (!rt_opt_noloop) {
    control = init_socket(port);
    sprintf(log_buf, "ROM is ready to rock on port %d.", port);
    log_string(log_buf);
    game_loop(control);
    CLOSE_SOCKET(control);

#ifdef _MSC_VER
    WSACleanup();
#endif
}
```

...to this:

```c
if (!rt_opt_noloop) {
    init_server(&server, port);
    sprintf(log_buf, "ROM is ready to rock on port %d.", port);
    log_string(log_buf);
    game_loop(&server);

    close_server(&server);
}
```

Here is where the real pay-off comes. I replace the guts of `game_loop()` like so, with a few lines of context to demonstrate where the changes begin and end:

```c
    // Main loop
    while (!merc_down) {
        PollData poll_data = { 0 };

        poll_server(server, &poll_data);

        // New connection?
        if (has_new_conn(server, &poll_data)) 
            handle_new_connection(server);

        process_client_input(server, &poll_data);

        // Autonomous game motion.
        update_handler();

        // Output.
        process_client_output(&poll_data);

        /*
         * Synchronize to a clock.
```

I replaced 117 lines of code with about a dozen, and have removed all traces of net code from the core game logic.

Now I can trim the includes in `main.c`:

```c
#include "merc.h"

#include "benchmark.h"
#include "comm.h"
#include "strings.h"

#ifdef _MSC_VER
#include <stdint.h>
#include <io.h>
#else
#include <stdlib.h>
#include <string.h>
#include <sys/time.h>
#include <unistd.h>
#ifndef _XOPEN_CRYPT
#include <crypt.h>
#endif
#endif
```

With this, I'm now ready to make new client creation an asynchronous operation, as the code that does so is now in a discrete function that can be parallelized.

## Asynchronous `accept()` Via Multi-Threading

Because (again, forecasting) SSL's handshake takes an indeterminate amount of time _and_ blocks execution, I have to ensure that operation is asynchronous. Furthermore, I need the results to be visible to the rest of the application, which means forking a separate process is not an option. Therefore, I need to use threads.

As of C11 there is a standard thread library. Unfortunately, it is 1) not robust and 2) not supported on MSVC. Therefore, I will need to implement two different threading solutions: native Win32 threads in MSVC and `pthreads` for everything else.

> There exist Win32 ports of `pthreads`, but I prefer to keep external dependencies to a minimum. Bringing in OpenSSL later will be more than enough work for me, thank you very much.

### Setting Up `pthreads`

Win32 will have multi-threading out of the box. For Linux/Cygwin, I need to bring in the `pthreads` library.

I add this near the top of `CMakeLists.txt`:

```cmake
if (NOT MSVC)
    set(THREADS_PREFER_PTHREAD_FLAG ON)
    find_package(Threads REQUIRED)
endif()
```

This defines `pthreads` as my threading library of choice, and then _requires_ it. 

Near the end of the file, in the linker options, I add threading to the non-MSVC option:

```cmake
if (NOT MSVC)
    target_link_libraries(rom crypt Threads::Threads)
```

The MSVC options are left unchanged.

### Implementing Threading

In `comm.h`, I set a maximum number of concurrent handshakes:

```c
#define MAX_HANDSHAKES 10
```

This is an arbitrary number. The space for them is preallocated, and do not take up overmuch room when there are very few TLS handshakes actually happening.

Later in `comm.h` I replace the definition of `init_descriptor()` to this:

```c
void handle_new_connection(SockServer* server);
```

I'm not actually removing `init_descriptor()`, but I am going to make it my internal async function, and `handle_new_connection()` will be the function that is called to spawn it.

That means I need to change this line in `main.c`, in `game_loop()`:

```c
init_descriptor(server);
```

...to this:

```c
handle_new_connection(server);
```

The rest of the changes will all occur in `comm.c`.

First, near the top of `comm.c`, just after global variable declarations, I add a new status for threads:

```c
typedef enum {
    THREAD_STATUS_NONE,
    THREAD_STATUS_STARTED,
    THREAD_STATUS_RUNNING, 
    THREAD_STATUS_FINISHED,
} ThreadStatus;
```

Because I defined a limited number of handshakes, I need to keep track of when they are finished so their "slot" can be reused. Each of these statuses also controls what operations can be performed in that slot. For instance, if any thread is listed as "started", I don't spawn any new threads, yet. This is because, until `accept()` is called, the listener socket will think it still has a new connection. When the status is set to "running", it's free to consider new connections.

Next I need to add some macros to reduce boilerplate on my async function:

```c
#ifdef _MSC_VER
#define INIT_DESC_RET DWORD WINAPI
#define INIT_DESC_PARAM LPVOID
#define THREAD_ERR -1
#else
#define INIT_DESC_RET void*
#define INIT_DESC_PARAM void*
#define THREAD_ERR (void*)-1
#endif
```

The `init_descriptor()` function will need a different signature between Win32 and `pthreads`. The different return types need a different cast.

Now I define the data that is used to keep track of each thread:

```c
typedef struct new_conn_thread_t {
#ifdef _MSC_VER
    HANDLE thread;
    DWORD id;
#else
    pthread_t id;
#endif
    ThreadStatus status;
} NewConnThread;
```

Both platforms will have a `ThreadStatus`, but otherwise require different types.

One nice thing, though, is that they can share the same data payload (which is passed to the asynchronous function as a parameter):

```c
typedef struct thread_data_t {
    SockServer* server;
    int index;
} ThreadData;
```

I pass in `server` so the async function has access to the listener socket (and, in the future, the SSL context). It also uses `index` to store that handshake's slot for updating the thread status.

The slots are defined thusly:

```c
NewConnThread new_conn_threads[MAX_HANDSHAKES] = { 0 };
ThreadData thread_data[MAX_HANDSHAKES] = { 0 };
```

Next I want to update `init_descriptor()`:

```c
static INIT_DESC_RET init_descriptor(INIT_DESC_PARAM lp_data)
{
```

> I made this method `static`, as it no longer needs to be accessed from another TU. Only functions in this TU can call it. This tidies up the linking and prevents spurious outside calls. Generally, it's a good idea.

In `init_descriptor()`, just below local declarations, I grab the `ThreadData` and `SockServer` from `lp_data`:

```c
ThreadData* data = (ThreadData*)lp_data;
SockServer* server = data->server;
```

I set the thread status to "running" just before calling `accept()`:

```c
    new_conn_threads[thread_data->index].status = THREAD_STATUS_RUNNING;
    if ((desc = accept(server->control, (struct sockaddr*)&sock, &size)) < 0) {
```

Note that by the time execution gets to this point, the status will already be "started", but I'll get to that.

There are a number of bare `return`s for error conditions. These need to be replaced, as this is no longer a `void` function:

```c
return THREAD_ERR;
```

The bare `return` at the end of the function, however, is replaced by thread clean-up.

```c
    new_conn_threads[thread_data->index].status = THREAD_STATUS_FINISHED;

#ifndef _MSC_VER
    pthread_exit(NULL);
#endif

    return 0;
```

> Guess what? There's a bug in my code that didn't pop up in my testing, but it needs to be handled: in the event of an error, we need to to more than returning `THREAD_ERR`; we also need to set the thread status to "finished" and call `pthread_exit()` for non-MSVC. But that's more boiler-plate than I prefer.
>
> How would YOU fix it?

### New Function: `handle_new_connection()`

Here's the meat and potatoes: this function spawns `init_descriptor()` on a new thread:

```c
void handle_new_connection(SockServer* server) {
    int new_thread_ix = -1;
```

I'll need to sift through the pile of handshake slots to find and open one. If I don't, then `new_thread_ix` will remain `-1`.

```c
    for (int i = 0; i < MAX_HANDSHAKES; i++) {
        if (new_conn_threads[i].status == THREAD_STATUS_STARTED) {
            // We are still processing a handshake. Do nothing.
            new_thread_ix = -2;
        }

        if (new_conn_threads[i].status == THREAD_STATUS_FINISHED) {
#ifdef _MSC_VER
            CloseHandle(new_conn_threads[i].thread);
#endif
            new_conn_threads[i].id = 0;
            new_conn_threads[i].status = THREAD_STATUS_NONE;
        }

        if (new_conn_threads[i].status == THREAD_STATUS_NONE
            && new_thread_ix == -1)
            new_thread_ix = i;
    }
```

Each thread sets its own status to "finished." This indicates that the slot is open and available. For MSVC, it's also a call to clean up the handle. Arguably, this should be done on tick, not on new connection. I personally don't think it's that big of a deal, but I leave that as an exercise to the reader.

Note the magic number `-2`. I'm using that to indicate that one of the threads has started, but has not accepted, yet. During this period, the master socket will have a "read" flag for an incoming connection. This could lead to multiple threads for the same connection!

I take care of that, next:

```c
    if (new_thread_ix == -1) {
        // New threads are maxed out.
        perror("Not ready to receive new connections, yet.");
        return;
    }
    else if (new_thread_ix == -2) {
        // Still spinning up a previous thread.
        return;
    }
```

The `-1` means there are no open slots. `-2` indicates that the master socket is still waiting for an `accept()`. Either way, I don't want to spawn a new `accept()` thread.

```c
    thread_data[new_thread_ix].index = new_thread_ix;
    thread_data[new_thread_ix].server = server;

    new_conn_threads[new_thread_ix].status = THREAD_STATUS_STARTED;
```

Next I populate the data payload that is passed to `init_descriptor()` as `lp_data`. This is the data I used in that function to get the proper slot for updating the thread status. I also need `server` to grab the master socket.

Then I set the status to "started", which effectively claims the slot and prevents it from being used by another incoming connection until it is finished with the handshake.

Finally, I perform the actual thread creation:

```c
#ifdef _MSC_VER
    new_conn_threads[new_thread_ix].thread = CreateThread(
        NULL,                                   // default security attributes
        0,                                      // use default stack size  
        init_descriptor,                        // thread function name
        &thread_data[new_thread_ix],            // argument to thread function 
        0,                                      // use default creation flags 
        &(new_conn_threads[new_thread_ix].id)   // returns the thread identifier 
    );

    if (new_conn_threads[new_thread_ix].thread == NULL) {
        perror("handle_new_connection()");
        return;
    }
#else
    if (pthread_create(&(new_conn_threads[new_thread_ix].id), NULL, 
            init_descriptor, &thread_data[new_thread_ix]) != 0) {
        perror("handle_new_connection(): pthread_create()");
    }
#endif
}
```

All in all, it pretty simple. Now `handle_new_connection()` is complete, and ROM now has a completely ansychronous `accept()` procedure. That is going to prove crucial to getting TLS-secured communications in the next post.

## Final Administrivia

This is not related to multi-threading at all, but I need to mention it because it's in the repo at this step. There is field in both `CHAR_DATA` and `class_table` called `class`. This will be problematic if I ever decide to move to C++, and Visual Studio Intellisense already complains about it.

I renamed it to `ch_class`, and went on with my day.

([Here is the code](https://github.com/bfelger/rom/tree/4c6562a39cf4627b63398022c222b8bafb9aa731) with updates from this post.)

Next: [Part 9](pt-tls)

Copyright 2023, Brandon Felger
