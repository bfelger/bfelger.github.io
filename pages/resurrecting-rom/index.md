---
title:  "Resurrecting ROM"
date:   2023-05-06 20:30:00 -0500
categories: mud C
---

# Resurrecting ROM

This is a series of posts about reviving and old, favorite project of mine, a MUD last released in 1997 called Rivers of MUD (ROM).

1. [Resurrecting ROM &mdash; Getting it to Compile Under Linux With GCC](pt-1-compile-gcc) - Starting with Ubuntu and GCC 11.3, getting ROM to compile and run without any build errors or warnings. _[2023-05-06]_
2. [Resurrecting ROM &mdash; Getting it to Compile With Clang and Cygwin](pt-2-compile-clang) - Continuing Ubuntu and Clang 14, then adding support for Cygwin on Windows. _[2023-05-09]_
3. [Resurrecting ROM &mdash; Dusting Off the Cobwebs](pt-3-dusting-off-cobwebs) - Excising old, archaic platforms to reduce macro hell (so we can add it back in Part 5). _[2023-05-10]_
4. [Resurrecting ROM &mdash; Cross-Platform Compilation With CMake](pt-4-cmake) - "One build system to rule them all." Leaning into MSVC support for CMake/Ninja, and getting lighting-fast build times, to boot. Also, updating to the C2X Draft Standard. _[2023-05-11]_
5. [Resurrecting ROM &mdash; Compiling ROM With MSVC (Part 1)](pt-5-msvc) - Getting ROM to build on Windows without errors. _[2023-05-16]_
6. [Resurrecting ROM &mdash; Compiling ROM With MSVC (Part 2)](pt-6-msvc-2) - Getting rid of all the warnings and messages. _[2023-05-18]_
7. [Resurrecting ROM &mdash; Benchmarks and Unit Tests](pt-7-testing) - Adding better command-line support, arbitrary run location, and a framework for benchmarking. _[2023-05-22]_
8. [Resurrecting ROM &mdash; Preparing for Secure Sockets](pt-8-mt-sockets) - Refactoring net code and asynchronous `accept()` with threads. _[2023-08-01]_
9. [Resurrecting ROM &mdash; Secure Sockets with OpenSSL](pt-9-tls) - Encrypting traffic with TLS. _[2023-08-04]_
10. [Resurrecting ROM &mdash; Cross-Platform Password Hashing](pt-10-pwd-hash) - Finally getting rid of `crypt()`. _[2023-08-05]_

## Related Topics

- [Should I Convert This Ancient C Code to C++?](topic-01-cpp) - No. Or... _maybe_. _[2023-08-09]_ 