---
layout: default
title:  "Resurrecting ROM Pt. 7 - Benchmarks and Unit Tests"
date:   2023-05-18 20:00:00 -0500
categories: mud C
---

# Resurrecting ROM Pt. 7 &mdash; Benchmarks and Unit Tests

This post picks up where [Part 6](pt-6-msvc-2) left off. [Here is the code](https://github.com/bfelger/rom/tree/ac968f668dd3f5c96eee55d6e855641d6a8ba496) that resulted from the work.

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
                continue;
            }
            else if (!strcmp(argv[i], "-p")) {
                if (++i < argc) {
                    port_str = argv[i];
                    continue;
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

I add two new options to the command-line handler, as well as the top-level variable to hold the string argument:

```c
    char* area_dir_str = NULL;
```

```c
            else if (!strcmp(argv[i], "-a")) {
                if (++i < argc) {
                    area_dir_str = argv[i];
                    continue;
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

