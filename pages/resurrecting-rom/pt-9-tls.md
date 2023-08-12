---
layout: default
title:  "Resurrecting ROM Pt. 9 - Secure Sockets with OpenSSL"
date:   2023-08-04 20:00:00 -0500
categories: mud C
---

# Resurrecting ROM Pt. 9 &mdash; Secure Sockets with OpenSSL

This post picks up where [Part 8](pt-8-mt-sockets) left off. [Here is the code](https://github.com/bfelger/rom/tree/4c6562a39cf4627b63398022c222b8bafb9aa731) that resulted from that work.

This day in age, I don't feel too good about running a server where users transmit credentials in the plain text, to be read by the entire world. I imagine there is a not insignificant number of people that use the same password for their bank and they do for their level 50 wizard.

It's time to bring ROM into the modern era and secure it with encryption. To do that, I'm going to implement TLS with OpenSSL.

## OpenSSL v3

The first thing I need is OpenSSL _with_ dev libraries. There are two options available (at the time of this writing): v1.1 and v3. Version 1.1 is still popular, but at EOL. I'm going to use v3, but the prevalence of v1.1 will cause problems for me on Windows (as you will see soon).

> Don't confuse OpenSSL v3, the _software package_ with SSL v3, the _defunct encryption protocol_. I will be using OpenSSL v3 to implement the TLS 1.3 protocol.

Now, where do I get OpenSSL v3? Strictly speaking, _there are no official binaries_. While every Linux distribution (and Cygwin) provide binaries for OpenSSL, it is recommended (for the sake of security) that you build it yourself. Win32 ports for MSVC are harder to come by, and originate from dodgier sites. 

Whether you choose to roll your own or not is up to you. In my opinion, OpenSSL is an easy library to build, but not necessarily easy to _use_, so I'm going to have to do the hard work, anyway.

To build OpenSSL v3, you need two tools: Perl 5 and `make`. Both are easy to get in Linux and Cygwin, but for Win32 you will need to source them.

> A word of warning: many implementations of Perl 5 (such as Strawberry Perl for Win32) come with OpenSSL 1.1, and will place it on your PATH, making it impossible for ROM to properly find the OpenSSL v3 libs.
>
> After building OpenSSL v3, I had to manually place its `bin` and `lib` directories higher in priority.

Building OpenSSL is largely an automated process. Do not skimp on the tests!

## Laying the Groundwork

In `CMakeLists.txt`, I add the following near the top of the file:

```cmake
set(OPENSSL_USE_STATIC_LIBS TRUE)
find_package(OpenSSL REQUIRED)
```

I also modify the linker options at the bottom of the file:

```cmake
if (NOT MSVC)
    target_link_libraries(rom crypt Threads::Threads OpenSSL::SSL)
else()
    target_link_libraries(rom OpenSSL::Crypto OpenSSL::SSL OpenSSL::applink)
endif()
```

> Hey, notice my old friend `crypt` there? Well, now that I have OpenSSL, I think its days are numbered. Now that I have a cross-platform, FIPS-compliant encryption solution, I am going to remove it soon.

After regenerating my CMake cache, I'm ready to dig in and integrate OpenSSL!

Except in MSVC.

_&lt;/sad-trombone&gt;_

### Configuring Visual Studio (Win32)

There a number of settings I need to manually set in my CMake settings (under `Advanced`; some may actually be automatically set, but not all. I have to verify them):

| Setting | Value |
| :--- | :--- |
| LIB_EAY_RELEASE | _&lt;OpenSSLDir&gt;_/lib/libcrypto.lib |
| OPENSSL_APPLINK_SOURCE | _&lt;OpenSSLDir&gt;_/include/openssl/applink.c |
| OPENSSL_INCLUDE_DIR | _&lt;OpenSSLDir&gt;_/include |
| SSL_EAY_DEBUG | _&lt;OpenSSLDir&gt;_/lib/libssl.lib |
| SSL_EAY_RELEASE | _&lt;OpenSSLDir&gt;_/lib/libssl.lib |

> _&lt;OpenSSLDir&gt;_ is the directory wherever I installed OpenSSL as part of the build process.

### Generating Certificates

The last bit of administrivia is generating SSL certificates for ROM to use. At the top level directory, I create a "keys" subdirectory, and add a new file called `ssl_gen_keys`:

```bash
#!/bin/bash

for ARG in "$@" 
do
    if [[ ${ARG,,} == "clean" ]]
    then
        rm -f *.key
        rm -f *.csr
        rm -f *.crt
        rm -f *.pem
    else
        echo "Unknown argument '${ARG,,}'."
        exit
    fi

done

if [ ! -f "rom-mud.key" ] || [ ! -f "rom-mud.csr" ]; then
    echo "Creating private key and signing request."
    openssl req -newkey rsa:2048 -nodes -keyout rom-mud.key -out rom-mud.csr
fi

if [ ! -f "rom-mud.crt" ]; then
    echo "Creating self-signed certificate."
    openssl x509 -signkey rom-mud.key -in rom-mud.csr -req -days 365 -out rom-mud.crt
fi

if [ ! -f "rom-mud.pem" ]; then
    echo "Creating PEM file."
    openssl x509 -in rom-mud.crt -out rom-mud.pem
fi

echo "Server's self-signed cert:"
openssl x509 -in rom-mud.pem -noout -text
```

> Remember to `chmod +x` this file to make is executable in Linux.

Before I can use SSL sockets, I need to create a certificate and a public key. This script performs the entire process for me; all I have to do is run:

```
./ssl_gen_keys
```

This script is interactive: it asks me questions that are required to generate the certificates. I could have written a script that just auto-populates dummy values, but that is not the correct way of doing it.

When asked for the domain name, I _must_ use the server name as I will be connecting to it. Hostname validation is an essential part of secure TLS communications. In testing (and local dev) I have no problem using my local box name. But for an internet-visible MUD, I would have to use the FQDN (fully-qualified domain name). Clients will be validating that it matches the server they connected to.

The password you enter doesn't really matter.

> This script can be easily converted to a simple 4-line batch file for Win32. In my case, I ran this script in Cygwin to generate the necessary files, which work regardless of platform.

## Code Changes for TLS

To make ROM aware of the certificate and private key, I add a reference to them with the other directories listed in `merc.h`:

```c
#ifndef USE_RAW_SOCKETS
#define CERT_FILE       "../keys/rom-mud.pem"
#define PKEY_FILE       "../keys/rom-mud.key"
#endif
```

I went back and forth on `USE_RAW_SOCKETS`, and whether I even wanted that to be a thing. In the end, I decided that leaving the ability to compile with bare telnet would be useful to anyone that preferred SSH tunnelling over TLS.

Depending on how hairy things get when I implement MSSP later, I may change my mind on this. For now, it stays.

> To compile with `USE_RAW_SOCKETS` enabled, add `-DUSE_RAW_SOCKETS` to the compiler flags in `CMakeLists.txt`, and remove references to SSL. I'm sure there is a more robust way to handle this in configuration, but it's not a priority, right now.

### Updating Client and Server Structures

Believe it or not, there is not a whole lot of work to do; the work I did in the last part made it very easy for me to integrate SSL into the current code.

Other than listing the certificates in `merc.h`, all code for SSL will go into `comm.c`.

To start with, I need to update `SockServer` and `SockClient`. I add a new header to `comm.h`:

```c
#ifndef USE_RAW_SOCKETS
#include <openssl/ssl.h>
#endif
```

Next I add a new field to `SockClient`:

```c
typedef struct sock_client_t {
    SOCKET fd;
#ifndef USE_RAW_SOCKETS
    SSL* ssl;
#endif
} SockClient;
```

The `SSL` object is the interface through which I perform TLS I/O. Note that I am still using `fd` to poll the client; even though traffic is encrypted, it still uses bare-metal sockets under the covers. All of my polling code still works.

I also add a new field to `SockServer`:

```c
typedef struct sock_server_t {
    SOCKET control;
#ifndef USE_RAW_SOCKETS
    SSL_CTX* ssl_ctx;
#endif
} SockServer;
```

The SSL context (`SSL_CTX`) will hold all the information needed to create and handle new SSL clients. As with client sockets, I will still make use of the master socket (`control`).

### Updating `close_server()`

The new structures I added have to be cleaned up. I don't want that to be an after thought, so I will take care of that, first.

I add some necessary headers to `comm.c`:

```c
#ifndef USE_RAW_SOCKETS
#include <openssl/err.h>
#include <openssl/pem.h>
#include <openssl/ssl.h>
#endif
```

I then add an SSL tear-down to `close_server()`, just after `CLOSE_SOCKET()`:

```c
    CLOSE_SOCKET(server->control);

#ifndef USE_RAW_SOCKETS
    SSL_CTX_free(server->ssl_ctx);
#endif
```

### Properly Cleaning Up In `init_server()`

I made a boo-boo in the last part. I added this nice function `close_server()` to close the socket _and_ de-initialize WinSock, and only used it in `game_loop()`. Meanwhile, `init_descriptor()` is still using `CLOSE_SOCKET()` on `server->control`.

I take the opportunity now to fix that by replacing several instances of:

```c
CLOSE_SOCKET(fd);
exit(1);
```

...with:

```c
close_server(server);
exit(1);
```

### New Function: `close_client()`

This is arguably an oversight from the last post, but now I need to perform a similar SSL clean-up for clients:

```c
void close_client(SockClient* client)
{
#ifndef USE_RAW_SOCKETS
    if (client->ssl) {
        SSL_shutdown(client->ssl);
        SSL_free(client->ssl);
    }
#endif
    CLOSE_SOCKET(client->fd);
}
```

Anywhere I previously called:

```c
CLOSE_SOCKET(client.fd);
```
```c
CLOSE_SOCKET(d->client.fd)
```

...I change to (respectively):

```c
close_client(&client);
```
```c
close_client(&d->client);
```

Now I'm fairly sure I've covered my bases on clean up.

### New Function: `init_ssl_server()`

This function is where I configure the SSL server, set TLS options, and select certificates. I will describe it here in parts.

As with the other code, I guard the function:

```c
#ifndef USE_RAW_SOCKETS
void init_ssl_server(SockServer* server)
{
    log_string("Initializing SSL server:");
```

Because I already added everything I need to `SockServer`, I don't need much else in the function signature.

```c
    SSL_library_init();
    OpenSSL_add_all_algorithms();
```

The first bit is boilerplate. The second bit is probably overkill: it enables all encryption algorithms. I could have just enabled the one used in my certificate. Perhaps I'll revisit this; perhaps not.

```c
    if ((server->ssl_ctx = SSL_CTX_new(TLS_server_method())) == NULL) {
        perror("! Unable to create SSL context");
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }
    else {
        printf("* Context created.");
    }
```

Here I set the SSL context, and set the _method_ as TLS _server_. This means that this context can only be used for TLS, and only for accepting connections.

SSL has its own functions for handling errors and converting error codes to strings. `ERR_print_errors_fp()` is a nice helper function to take care of all of that, for me.

```c
    char cert_file[256];
    sprintf(cert_file, "%s%s", area_dir, CERT_FILE);

    char pkey_file[256];
    sprintf(pkey_file, "%s%s", area_dir, PKEY_FILE);
```

I grab the file names for my certificate and private key, which I then feed into the next parts.

```c
    if (SSL_CTX_use_certificate_file(server->ssl_ctx, cert_file, 
            SSL_FILETYPE_PEM) <= 0) {
        fprintf(stderr, "! Failed to open SSL certificate file %s\n", cert_file);
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }
    else {
        printf("* Certificate %s loaded.\n", cert_file);
    }
```

This function loads the certificate directly from the PEM file and assigns it to the SSL context object.

```c
    if (SSL_CTX_use_PrivateKey_file(server->ssl_ctx, pkey_file, 
            SSL_FILETYPE_PEM) <= 0) {
        fprintf(stderr, "! Failed to open SSL private key file %s\n", pkey_file);
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }
    else {
        printf("* Private key %s loaded.\n", cert_file);
    }
```

I then perform a similar operation with the private key, loading it from its PEM file and assigning it to the SSL context.

```c
    if (SSL_CTX_check_private_key(server->ssl_ctx) <= 0) {
        perror("! Certificate/pkey validation failed.\n");
        ERR_print_errors_fp(stderr);
        exit(EXIT_FAILURE);
    }
    else {
        printf("* Cert/pkey validated.\n");
    }
```

If I messed up certificate creation, this is where I would find out. This code validates the private key against the certificate to ensure they match. This is essential, because otherwise ROM would not be able to encrypt/decrypt client traffic.

```c
    // Only allow TLS
    SSL_CTX_set_options(server->ssl_ctx, SSL_OP_ALL | SSL_OP_NO_SSLv2 
        | SSL_OP_NO_SSLv3);
}
#endif
```

I now finish up `init_ssl_server()` by disabling SSLv2 and v3, leaving only TLS as an available option for the server.

> Why? Because SSLv3 was busted so soon after its release (1996) that TLS was created just to cover its flaws just three years later. By 2014, it had been so gutted by vulnerabilities like POODLE that pretty much everything dropped support for it. It has to be actively disabled because if it's even an _option_, a nefarious client can negotiate a _downgrade_ from TLS, which pretty much gives them the keys to the kingdom.
>
> SSL v2 was replaced by v3 only a year after its release, so that really isn't an option, either.

### Updating `init_server()`

There isn't anything I really have to do in `init_server()`, it's just a convenient place to add a call to `init_ssl_server()`. I add this near the top of the function:

```c
#ifndef USE_RAW_SOCKETS
    init_ssl_server(server);
#endif
```

The rest of my socket code stays the same, as the magic happens during the handshake.

### Updating `init_descriptor()`

The hard work from Part 8 pays off right here. The `accept()` process, being already asynchronous, can take as long as it wants. All I have to do is add this just after the `accept()`:

```c
#ifndef USE_RAW_SOCKETS
    client.ssl = SSL_new(server->ssl_ctx);
    SSL_set_fd(client.ssl, (int)client.fd);

    if (SSL_accept(client.ssl) <= 0) {
        ERR_print_errors_fp(stderr);
        close_client(&client);
        return THREAD_ERR;
    }
#endif
```

I use the server's SSL context to create a new client object, and assign the already-accepted client socket to it. `SSL_accept()` performs the entire TLS handshake, including the exchange of certificates.

Note the case of `client.fd` to `int`. This is needed, even in MSVC, as SSL assumes that the client file descriptor is an `int` in all cases.

All that remains is updating the socket I/O.

### Updating `read_from_descriptor()`

Currently, I'm not using the error codes from the SSL function (because of `ERR_print_errors_fp()`'s catch-all nature), but I can't guarantee that that will always be the case. I define a placeholder for it at the top of the function:

```c
#ifndef USE_RAW_SOCKETS
    int ssl_err;
#endif
```

I then replace the `read()`/`recv()` call:

```c
#ifndef USE_RAW_SOCKETS
        size_t nRead;
        if ((ssl_err = SSL_read_ex(d->client.ssl, d->inbuf + iStart,
                sizeof(d->inbuf) - 10 - iStart, &nRead)) <= 0) {
            ERR_print_errors_fp(stderr);
            return false;
        }
#else
        int nRead;
#ifdef _MSC_VER
        nRead = recv(d->client.fd, d->inbuf + iStart, 
            (int)(sizeof(d->inbuf) - 10 - iStart), 0);
#else
        nRead = read(d->client.fd, d->inbuf + iStart,
            sizeof(d->inbuf) - 10 - iStart);
#endif
#endif // !USE_RAW_SOCKETS
```

Overall, the logic is the same. Note that `nRead` is defined as `size_t`, here.

That's it for input. `SSL_read_ex()` handles all the decryption for me.

### Updating `write_to_descriptor()`

Like before, I want to grab SSL error codes just in case I decide to use them, later:

```c
#ifndef USE_RAW_SOCKETS
    int ssl_err;
#endif
```

I also similarly replace the `send()`/`write()` call:

```c
#ifndef USE_RAW_SOCKETS
        if ((ssl_err = SSL_write_ex(d->client.ssl, txt + iStart, nBlock, &nWrite)) <= 0) {
            ERR_print_errors_fp(stderr);
            return false;
        }
#else
#ifdef _MSC_VER
        if ((nWrite = send(d->client.fd, txt + iStart, nBlock, 0)) < 0) {
            PrintLastWinSockError();
#else
        if ((nWrite = write(d->client.fd, txt + iStart, nBlock)) < 0) {
#endif
            perror("Write_to_descriptor");
            return false;
        }
#endif // !USE_RAW_SOCKETS
```

As with `SSL_read_ex()`, `SSL_write_ex()` handles its side of the business entirely, encrypting data using the private key and sending it off the the client.

And with that, everything should build, and I am now able to test the server.

## Testing TLS Connections

I boot up my client of choice, Mudlet, and check "Secure" to force it to use TLS. I also have to set "Server Address" to the same value I used in creating my self-signed certificate. When prompted, I also need to check the option "Accept self-signed certificates".

After accepting the certificate transaction, the TLS handshake is now complete, and I am graced with this:

```
THIS IS A MUD BASED ON.....

                                ROM Version 2.4 beta

               Original DikuMUD by Hans Staerfeldt, Katja Nyboe,
               Tom Madsen, Michael Seifert, and Sebastian Hammer
               Based on MERC 2.1 code by Hatchet, Furey, and Kahn
               ROM 2.4 copyright (c) 1993-1998 Russ Taylor

By what name do you wish to be known?
```

All communications between ROM and the client are now secure. This is big step up from how I left things with my old mud 20 years ago.

## Housekeeping

Since I'm pretty firmly set on CMake for the duration of this project, I removed `Makefile` and `Makefile.clang` from the `src` folder. You can easily keep and maintain them, if you prefer.

## Coda

There is just one more thing I have to do before I can consider ROM Resurrected "complete". I have to finally slay the beast that has haunted me from the very outset: `crypt()`.

([Here is the code](https://github.com/bfelger/rom/tree/4f484d0917dddc94473e80fe5927b7826a21b3af) with updates from this post.)

Next: [Part 10](pt-pwd-hash)

Copyright 2023, Brandon Felger
