---
title:  "Resurrecting ROM"
date:   2023-05-06 20:30:00 -0500
categories: mud C ROM Mud98
---

# Resurrecting ROM

Long ago, in the olden days of yore (1998), I became an "Implementor" and coder for a MUD called "Cities of M'Dhoria", based on "Rivers of MUD" 2.3 (aka ROM). The lead Implementor (Jeremy aka "Sativa") brought me on despite the fact that I had only yet taken Computer Programming 201, and knew only a smattering of C++. This project was my first real taste of old-school C, and taught me more about coding than I ever learned in school. Also, I think I made some really cool stuff.

It has been two decades since I last touched ROM, and in that time I've written a lot of C code as a professional software developer. After recently running across the [/r/MUD](https://www.reddit.com/r/MUD/) subreddit, my fond remembrance of hacking on that old ROM code got me wondering if I could do a better job of this this time around.

This series of posts is a journal of my long-term project to "resurrect" ROM and turn it into a modern C project, adhering to current standards and software design principles. The result is Mud98 (an homage to ROM's final release date).

The for goal for Mud98 to be a turn-key baseline that MUD builders and enthusiasts can use as a starting point for their own projects.

## Mud98 Git Repository

The source code for Mud98 can be found here: https://github.com/bfelger/Mud98

## Posts

1. [Resurrecting ROM &mdash; Getting it to Compile Under Linux With GCC](pt-1-compile-gcc) - Starting with Ubuntu and GCC 11.3, getting ROM to compile and run without any build errors or warnings. _[2023-05-06]_
2. [Resurrecting ROM &mdash; Getting it to Compile With Clang and Cygwin](pt-2-compile-clang) - Continuing Ubuntu and Clang 14, then adding support for Cygwin on Windows. _[2023-05-09]_
3. [Resurrecting ROM &mdash; Dusting Off the Cobwebs](pt-3-dusting-off-cobwebs) - Excising old, archaic platforms to reduce macro hell (so we can add it back in Part 5). _[2023-05-10]_
4. [Resurrecting ROM &mdash; Cross-Platform Compilation With CMake](pt-4-cmake) - "One build system to rule them all." Leaning into MSVC support for CMake/Ninja, and getting lighting-fast build times, to boot. Also, updating to the C2X Draft Standard. _[2023-05-11]_
5. [Resurrecting ROM &mdash; Compiling ROM With MSVC (Part 1)](pt-5-msvc) - Getting ROM to build on Windows without errors. _[2023-05-16]_
6. [Resurrecting ROM &mdash; Compiling ROM With MSVC (Part 2)](pt-6-msvc-2) - Getting rid of all the warnings and messages. _[2023-05-18]_
7. [Resurrecting ROM &mdash; Benchmarks and Unit Tests](pt-7-testing) - Adding better command-line support, arbitrary run location, and a framework for benchmarking. _[2023-05-22]_
8. [Resurrecting ROM &mdash; Preparing for Secure Sockets](pt-8-mt-sockets) - Refactoring net code and asynchronous `accept()` with threads. _[2023-08-01]_
9. [Resurrecting ROM &mdash; Secure Sockets with OpenSSL](pt-9-tls) - Encrypting traffic with TLS. _[2023-08-04]_
10. [Resurrecting ROM &mdash; Cross-Platform Password Hashing](pt-10-pwd-hash) - Finally getting rid of `crypt()`. _[2023-08-12]_

## Related Topics

- [Should I Convert This Ancient C Code to C++?](topic-01-cpp) - No. Or... _maybe_. _[2023-08-09]_ 