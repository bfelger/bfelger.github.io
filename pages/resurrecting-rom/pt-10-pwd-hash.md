---
layout: default
title:  "Resurrecting ROM Pt. 10 - Cross-Platform Password Hashing"
date:   2023-08-12 20:00:00 -0500
categories: mud C
---

# Resurrecting ROM Pt. 10 &mdash; Cross-Platform Password Hashing

This post picks up where [Part 9](pt-9-tls) left off. [Here is the code](https://github.com/bfelger/Mud98/tree/4f484d0917dddc94473e80fe5927b7826a21b3af) that resulted from that work.

If you recall from [Part 5](pt-5-msvc), I have been "riding dirty" with clear-text passwords in player files due to a lack of a `crypt()` implementation. But now that I have cross-platform TLS implemented via OpenSSL, I also now have, by happenstance, a cross-platform _hashing_ solution.

It's time to fix and unify password hashing on all platforms.

## Securing Passwords with OpenSSL

The field `pwd` in `PC_DATA` holds the player's password. If `crypt()` is available, it is hashed with that particular platform's own POSIX implementation. Sometimes it's good. Sometimes it's bad. Generally, modern OS's have a good enough implementation. Unfortunately, the Win32 API doesn't want me to have easy access to hashing, so no `crypt()`, there.

Thankfully, I am fairly sure of OpenSSL's SHA-256 digests (the modern word for the output of hashing algorithms ["modern" still being older then ROM itself]), so I can use that for Win32. However, I also have some OCD, so I want all my platforms to be unified in this regard.

Once I'm done, I can rip out all traces of `crypt()`, its includes, and its attendant preprocessor directives.

### New file: `digest.h`

This new file is fairly straightforward; it's just function definitions:

```c
////////////////////////////////////////////////////////////////////////////////
// digest.h
//
// Create and manage SHA256 hashes with OpenSSL
////////////////////////////////////////////////////////////////////////////////

#pragma once
#ifndef ROM__DIGEST_H
#define ROM__DIGEST_H

#include "merc.h"

void bin_to_hex(char* dest, const uint8_t* data, size_t size);
bool create_digest(const char *message, size_t message_len, 
    unsigned char **digest, unsigned int *digest_len);
void decode_digest(PC_DATA* pc, const char* hex_str);
void free_digest(unsigned char* digest);
void hex_to_bin(uint8_t* dest, const char* hex_str, size_t bin_size);
bool set_password(char* pwd, CHAR_DATA* ch);
bool validate_password(char* pwd, CHAR_DATA* ch);

#endif // !ROM__DIGEST_H
```

My Git repo has a different name for `create_digest()`. The naming of things is something I struggle with, as I like self-documenting code, but often find accuracy and brevity to be at cross-purposes. In this, case, however, I was just _wrong_. I originally named this function `digest_message()`.

A "digest" is the result of a "hash"; it's a noun, not a verb. Unlike previous posts, I am writing this one "after the fact"; I'm already a post ahead on changes, and this bad name is already baked into the Git repo. Nevertheless, I'm changing it _now_.

Note that there are no external structures to this. There is going to be a new `digest` and `digest_len` field that I could have combined into an aggregate to lower the number of required function arguments. I'll leave that as an exercise to the reader.

### New file: `digest.c`

This is where all the work is going to happen, so I'm going to take it in chunks. Here's the boilerplate:

```c
////////////////////////////////////////////////////////////////////////////////
// digest.c
//
// Create and manage SHA256 hashes with OpenSSL
////////////////////////////////////////////////////////////////////////////////

#include "digest.h"

#include <string.h>

#include <openssl/err.h>
#include <openssl/evp.h>
```

I have used `openssl/err.h` before, for `ERR_print_errors_fp()`. But `openssl/evp.h` is new, here. "EVP" in this context is short for "envelope"; which in traditional parlance is a physical structure that encapsulates a message, and seals it away from anyone except the intended recipient. A digest is merely the digital, cryptographic equivalent.

This library was actually fairly difficult to find documentation for; one of the dangers of Stackoverflow is that those highly upvoted, accepted answers never get retracted, even when the proffered solution is for an old, deprecated API. Most of the internet still refers to EVP's predecessors, a mishmash of different lower-level API's that dealt directly with specific DSA or RSA modules. EVP unifies these disparate features with other crypto functions into one cohesive interface. If there is ever a future problem with, say, SHA-256 (cross your fingers), I can fairly easily swap it out.

I present the functions in order of importance, not necessarily in the other I put them in `digest.c`.

### New function: `create_digest()`:

Here's the signature for this new function:

```c
bool create_digest(const char* message, size_t message_len, 
    unsigned char** digest, unsigned int* digest_len)
{
```

The first two fields, `message` and `message_len`, pretty obviously deal with the string I want hashed. The latter two are "out" parameters*; I want an unallocated pointer I can fill with a pointer to a byte array, and the address of an `int` to report the number of bytes written to that array.

> \*An "out" parameter is one where the function is given an _address_ to the value to fill, rather than being expected to provide it on return. In this case, `unsigned char**` is a pointer to `unsigned char*` (a byte array). Calling code provides the address to fulfull the pointer like so:
> ```c
> unsigned char bytes*;
> unsigned int len;
> create_digest("a string", 8, &bytes, &len);
> ```
> In the same way that `&`_`<int>`_ fulfills a `int*`, so too does a `&`_`<char>*`_ fulfill a `char**`. This is called "double indirection", and is how a function can change an outside pointer's target address, rather than just the value _at_ the address.

Strictly speaking, I'm not thrilled that `unsigned char*` and `unsigned int` are being used instead of `uint8_t*` and `size_t`, respectively, but I didn't write OpenSSL (nor _could_ I), so I don't really get a say.

```c
    EVP_MD_CTX* mdctx;

    if ((mdctx = EVP_MD_CTX_new()) == NULL)
        goto digest_message_err;
```

> _"`goto`! `goto`! We got `goto` here! See? Nobody cares."_ &ndash; Nelly (paraphrased)

Similar to how OpenSSL handles secure sockets, I need to create a context for doing this work. Unlike `SSL_CTX`, however, I don't need to keep it around between digest requests.

Just as before, this is a resource that needs to be released when I'm done with it. Because I'm about to write a chain of fallible calls, I centralize the clean-up code in a single error handler with `goto`.

In another post, [Should I Convert My Ancient C Code to C++?](./topic-01-cpp.md), I present alternative ways of handling this resource clean-up by means of either RAII or exception-handling. For the most part, C++ obviates most of my use-cases for `goto` in C (as do other higher-level abstractive languages).

```c
    if (1 != EVP_DigestInit_ex(mdctx, EVP_sha256(), NULL))
        goto digest_message_err;
```

This is where I select the algorithm to use. This demonstrates the flexibility of EVP over the legacy API, where each algorithm had bespoke front-to-back implementations. If I want to swap out the algorithm, I need only do it in this one place.

The convention of placing immutable literals first is a safety mechanism against performing assignment instead of comparison. It's not a convention I make use of, myself; but since this code is adapted from OpenSSL's reference implementation with as little change as possible, it's here.

```c
    if ((*digest = (unsigned char*)OPENSSL_malloc(EVP_MD_size(EVP_sha256()))) == NULL)
        goto digest_message_err;
```

OpenSSL uses its own resource allocation. I could have created by own buffer and passed that in, but I figure this will give me the flexibility of choosing a different hash algorithm (which may require a different digest size) without worrying about the structure of the digest, itself.

This is another resource that needs to be cleaned up, but not until the digest is no longer needed. Since I'm using it to store passwords for user authentication, I need it available from when a user attempts to log in until they log out (because I need to re-save it, and give them the ability to change their password while they are logged in.)

```c
    if (1 != EVP_DigestFinal_ex(mdctx, *digest, digest_len))
        goto digest_message_err;
```

This is where the digest is actually persisted to the buffer, and the number of bytes written to it is reported. If this fails, there is nothing to report.

Oh, but there _is_ a memory leak. This is why journaling my code is useful; absent a code-reviewer, this is the best way catch my mistakes. Here's the _correct_ version of the above:

```c
    if (1 != EVP_DigestFinal_ex(mdctx, *digest, digest_len)) {
        free_digest(*digest);
        goto digest_message_err;
    }
```

> Like the change to `create_digest()`, this version is not in the Git repo linked in this post; but it _is_ in Lastest.

Now, would this ever come up? Theoretically, the hash could fail. But since the buffer is provided by OpenSSL itself, and is guaranteed to be the correct size, I am not very worried about it.

Nevertheless, it _does_ have to be changed.

```c
    EVP_MD_CTX_free(mdctx);
    return true;
```

At this point, all operations have complete successfully, and `create_digest()` has already successfully assigned its internally created buffer to whatever outside function called it.

All that remains is to clean up the context, and report success.

```c
digest_message_err:
    // C++ exceptions are looking pretty nice, right now...
    ERR_print_errors_fp(stderr);
    EVP_MD_CTX_free(mdctx);
    return false;
}
```

Here's the clean-up. It should be fairly straightforward.

Speaking of clean-up...

### New function: `free_digest()`

This one is pretty simple:

```c
void free_digest(unsigned char* digest)
{
    OPENSSL_free(digest);
}
```

The only reason for a one-liner like this function to exist is that I don't want to pollute the rest of my code with the specific implementation of my hash solution. Theoretically, I could one day replace OpenSSL with some hypothetical "LibreSSL", or my own handwritten implementation (spoiler alert: that is _never, ever_ happening).

### New function: `set_password()`

One of the first things a player does in ROM is select a password to go with their new character. If `crypt()` was available previously, this password was not clear-text at any level. This is the equivalent operation under the new regime:

```c
bool set_password(char* pwd, CHAR_DATA* ch)
{
    if (ch->pcdata->pwd_digest != NULL)
        free_digest(ch->pcdata->pwd_digest);

    if (!create_digest(pwd, strlen(pwd), &ch->pcdata->pwd_digest,
            &ch->pcdata->pwd_digest_len)) {
        perror("set_password: Could not get digest.");
        return false;
    }

    return true;
}
```

If this function is used to _reset_ a password, it will first free the old digest before creating one for the new password by called `create_digest()`. There is a preview here of the new fields in `PCDATA` to hold password digests.

### New function: `validate_password()`

Validating passwords is the entire _raison d'être_ of all this digest code. Here is the function ROM will use for authentication:

```c
bool validate_password(char* pwd, CHAR_DATA* ch)
{
    if (ch->pcdata->pwd_digest == NULL || ch->pcdata->pwd_digest_len == 0)
        return false;

    unsigned char* new_digest;
    unsigned int new_digest_len;
    if (!create_digest(pwd, strlen(pwd), &new_digest, &new_digest_len)) {
        perror("validate_password: Could not get digest.");
        return false;
    }

    return ch->pcdata->pwd_digest_len == new_digest_len &&
        !memcmp(new_digest, ch->pcdata->pwd_digest, new_digest_len);
}
```

If the user has no password, there are bigger problems. There is no need to handle them _here_. After ensuring there is a password to compare against. I create a digest for the password proffered during login. Since SHA-256 is a deterministic hash, if the passwords are the same, their respective digests will be exactly identical. Furthermore, if they are different, there is _theoretically_ no chance of collision (same hash result).

Therefore, with two digests in hand, one for the stored password and another for prospective authentication, it is simply a matter of comparing the digests at the byte level. For this purpose, `memcmp()` works the same as `strcmp()`; a result of `0` indicates they are the same.

### New function: `bin_to_hex()`

The password digest is a byte array of non-ASCII characters. To effectively store them in what is otherwise a human-readable player file, I need to translate it to a human-readable format:

```c
void bin_to_hex(char* dest, const uint8_t* data, size_t size)
{
    for (int i = 0; i < size; ++i) {
        sprintf(dest, "%.2X", *data++);
        dest += 2;
    }
}
```

Using C-format specifiers to print out two-digit hex-codes per byte is hack, but it's fast and it works.

For my test password, `"password"`, this function writes the following string to the buffer passed into `dest`:

```
5E884898DA28047151D0E56F8DC6292773603D0D6AABBDD62A11EF721D1542D8
```

If someone were to get their hands on the player file for any given person, they could not reasonably decipher their password.

> Now, in such case, I have _much_ bigger problems to worry about, but at least Joe-Bob's universal password for ROM, Paypal, and Well Fargo is safe.

I deliberated over whether or not to put this function in `digest.c`, or in `strings.c`. I chose here, but if I end up needing to persist more binary data to disk, I'll move it.

### New function: `hex_to_bin()`

Going in the opposite direction is trickier; there is a corresponding `sscanf()` trick that won't work, as it will want to pad buffer with a null-terminator that the digest won't have room for. Therefore, I do it by hand:

```c
void hex_to_bin(uint8_t* dest, const char* hex_str, size_t bin_size)
{
    for (int i = 0; i < bin_size; ++i) {
        char ch1 = *hex_str++;
        uint8_t hi = (ch1 >= 'A') ? (ch1 - 'A' + 10) : (ch1 - '0');
        char ch2 = *hex_str++;
        uint8_t lo = (ch2 >= 'A') ? (ch2 - 'A' + 10) : (ch2 - '0');
        uint8_t byte = (hi << 4) | (lo & 0xF);
        *dest++ = (uint8_t)byte;
    }
}
```

An 8-bit byte can be described as a composition of a "high" 4-bit number followed by a "low" 4-bit number. Each 16-digit hex value represents these four bits. Here I assemble two `uint8_t`s for which my only concern is the lower 4-bits. I shift the high bits and OR them with a mask of the low bits. The result is a single `uint8_t`.

### New function: `decode_digest()`

Now I function that takes that decoded binary and populates a digest with it, as if it had come from `create_digest()`. This is how a player's password digest will be loaded from the player file for the purposes of authentication:

```c
void decode_digest(PC_DATA* pc, const char* hex_str)
{
    if (pc->pwd_digest != NULL)
        OPENSSL_free(pc->pwd_digest);

    pc->pwd_digest_len = EVP_MD_size(EVP_sha256());
    pc->pwd_digest = (unsigned char*)OPENSSL_malloc(pc->pwd_digest_len);

    hex_to_bin(pc->pwd_digest, hex_str, pc->pwd_digest_len);
}
```

Like `set_password()`, I don't want to blindly set a player's digest without cleaning up the old one if it exists, so check the digest for `NULL` and free it if it exists.

I grab the digest size for SHA-256, and use that to create the target buffer for the digest. I use `OPENSSL_malloc()` so that there is a single, canonical way to free the digest buffer, and I don't have to keep a spare value around to indicate which memory management scheme to use.

With the digest space allocated and assigned to the `PCDATA` record, I use `hex_to_bin()` to populate it from `hex_str`.

And with that, I am now done with `digest.c`.

### New digest field for `PCDATA`

In `merc.h`, in `struct pc_data`, I want to replace this:

```c
    char* pwd;
```

...with this:

```c
    unsigned char* pwd_digest;
    unsigned int pwd_digest_len;
```

The old field could _either_ hold an encrypted password if _crypt()_ is available, or a clear-text password if not. This new field has no such double-purpose. It is an `unsigned char*` and a `size_t` because, as I mentioned before, this is what EVP expects. 

## Authentication and New Accounts

When you first connect to ROM, you are asked to provide a name. Then you are asked for a password. For existing characters, this is a challenge; for new characters it's a new password that will subsequently be verified before creating the player account.

To do this, changes need to be made to account handling. First, I add the new digest header to `comm.c`:

```c
#include "digest.h"
```

### Validating old passwords

In `nanny()`, I can get rid of the `p` and `pwnew` local variables. Then I take care of password validation under the `CON_GET_OLD_PASSWORD` case by replacing this:

```c
if (strcmp(crypt(argument, ch->pcdata->pwd), ch->pcdata->pwd)) {
```

...with this:

```c
if (!validate_password(argument, ch)) {
```

This is a better approach, anyway, because the implementation details of password handling are hidden away. The old way necessitated "dummy" preprocessor directives for missing `crypt()`.

###  Setting passwords for new players

Later, under the `CON_GET_NEW_PASSWORD` case, I replace this:

```c
    pwdnew = crypt(argument, ch->name);
    for (p = pwdnew; *p != '\0'; p++) {
        if (*p == '~') {
            write_to_buffer(
                d,
                "New password not acceptable, try again.\n\rPassword: ", 0);
            return;
        }
    }

    free_string(ch->pcdata->pwd);
    ch->pcdata->pwd = str_dup(pwdnew);
```

with this:

```c
    if (!set_password(argument, ch)) {
        perror("nanny: Could not set password.");
        close_socket(d);
        return;
    }
```

The only reason my code isn't a one-liner is because my implementation, unlike the old `crypt()` implementation, assumes the hash may fail. It is highly unlikely, but I am not taking any chances. And if I can't hash the password, I can't let them continue in the login process.

Note that I removed the check for the tilde character (`~`). This is the delimiter for text fields in the player file, so in clear text it would be problematic. That is no longer a concern, as the tilde will by no means be present in the hex-string representation of its SHA256 hash.

The player is then asked to confirm the new password under the `CON_CONFIRM_NEW_PASSWORD`. I replace this:

```c
   if (strcmp(crypt(argument, ch->pcdata->pwd), ch->pcdata->pwd)) {
        write_to_buffer(d, "Passwords don't match.\n\r"
            "Retype password: ", 0);
```

...with this:

```c
    if (!validate_password(argument, ch)) {
        write_to_buffer(d, "Passwords don't match.\n\r"
            "Retype password: ", 0);
```

### Copying digest on reconnection

In `check_reconnect()` there's a bit of code that no longer makes sense:

```c
    free_string(d->character->pcdata->pwd);
    d->character->pcdata->pwd = str_dup(ch->pcdata->pwd);
```

If a character is disconnected and reconnects, their new `DESCRIPTOR_DATA` needs to take over for their old one, including assuming its old resources (in the before case, it's allocated password string).

I change this code to this:

```c
if (d->character->pcdata->pwd_digest != NULL)
    free_digest(d->character->pcdata->pwd_digest);
d->character->pcdata->pwd_digest = (unsigned char*)OPENSSL_memdup(
    ch->pcdata->pwd_digest, 
    ch->pcdata->pwd_digest_len);
d->character->pcdata->pwd_digest_len = ch->pcdata->pwd_digest_len;
```

> Yeah, I should have made a macro to this.

### Digest cleanup 

In `recycle.c`, I include `digest.h` and change this line in `free_pcdata*()`:

```c
free_string(pcdata->pwd);
```

...to this:

```c
free_digest(pcdata->pwd_digest);
```

### Player password change

There is an in-game command called `password` that allows a user to change their password at any time. To do so, they must provide both the old password (to be validated) and the new one (to be saved). To handle that, I add `digest.h` to `act_comm.c` and alter `do_password()`. Like in `nanny()` before, I delete the local variables `p` and `pwnew`. Then I change the validation from this:

```c
if (strcmp(crypt(arg1, ch->pcdata->pwd), ch->pcdata->pwd)) {
```

...to this:

```c
if (!validate_password(arg1, ch)) {
```

That was fairly trivial. But now I need to set the new password. The new password check has the same unnecessary tilde-check, so there is significant savings in code when I change this:

```c
    /*
     * No tilde allowed because of player file format.
     */
    pwdnew = crypt(arg2, ch->name);
    for (p = pwdnew; *p != '\0'; p++) {
        if (*p == '~') {
            send_to_char("New password not acceptable, try again.\n\r", ch);
            return;
        }
    }
```

...to this:

```c
    if (!set_password(arg2, ch)) {
        perror("do_password: Could not get digest.");
    }
```

In this case, losing a password is pretty serious. I just noticed that (I seem to be catching a lot of bugs during this journaling process, sorry about that). I can fix that by adding transparent recovery for `set_password()` in `digest.h`:

```c
bool set_password(char* pwd, CHAR_DATA* ch)
{
    // If the operation fails, keep the old digest
    unsigned char* old_digest = ch->pcdata->pwd_digest;
    unsigned int old_digest_len = ch->pcdata->pwd_digest_len;

    // So create_digest() has no way to free the the old digest, yet (it 
    // doesn't, but what if that changes in the future? I don't want to have to
    // worry about it.
    ch->pcdata->pwd_digest = NULL;
    ch->pcdata->pwd_digest_len = 0;

    if (!create_digest(pwd, strlen(pwd), &ch->pcdata->pwd_digest,
            &ch->pcdata->pwd_digest_len)) {
        perror("set_password: Could not get digest.");

        ch->pcdata->pwd_digest = old_digest;
        ch->pcdata->pwd_digest_len = old_digest_len;
        return false;
    }

    // Ok, NOW we can throw it away.
    if (old_digest != NULL)
        free_digest(old_digest);

    return true;
}
```

That's a lot more robust, fault-tolerant, and most of all _recoverable_. The player's login should not be "bricked" by a bad password change.

> This code is in Latest, but not in the Git repo linked here.
>
> Tired of hearing that, yet?

## Loading and Saving Player Files

Now that I have this new field, I need to get it into, and out of, the otherwise clear-text player files.

All of this happens in `save.c`, which needs to include `digest.h` (naturally).

### Saving digests

This one is fairly simple. In `fwrite_char()`, change this code:

```c
fprintf(fp, "Pass %s~\n", ch->pcdata->pwd);
```

...to this:

```c
char digest_buf[256];
bin_to_hex(digest_buf, ch->pcdata->pwd_digest, ch->pcdata->pwd_digest_len);
fprintf(fp, "PwdDigest %s~\n", digest_buf);
```

There's my friendly helper function `bin_to_hex()`. It takes the ASCII hex-string representing the digest's byte-array and emits it to the player file as a tilde-delimited string.

I could have one-lined this by returning a static buffer, but I have a deep distaste for that mechanism (never mind that it's used extensively elsewhere in ROM). A better solution to make an `fwrite_digest()` or some such.

Note that the old password field is gone. MSVC is no longer "riding dirty", as I called it before, and is secure (or at least just enough for me not to feel bad about hosting a server with Joe-Bob's Wells Fargo password somewhere on the disk).

### Loading digests

In `load_char_obj()`, the initial `PCDATA` record is created with empty defaults. I change this:

```c
    ch->pcdata->pwd = str_dup("");
```

...to this:

```c
    ch->pcdata->pwd_digest = NULL;
    ch->pcdata->pwd_digest_len = 0;
```

That's pretty simple. But in `fread_char()`, things get a bit more difficult. I change these lines:

```c
            KEY("Password", ch->pcdata->pwd, fread_string(fp));
            KEY("Pass", ch->pcdata->pwd, fread_string(fp));
```

to this:

```c
#ifdef _MSC_VER
    // Legacy password support for platforms without crypt(). 
    // Read it, hash it, and throw it away.
    // Because Linux has crypt() available, this won't work for that
    // platform. 
    if (!str_cmp(word, "Password") || !str_cmp(word, "Pass")) {
        char* pwd = fread_string(fp);
        set_password(pwd, ch);
        free_string(pwd);
        fMatch = true;
        break;
    }
#endif

    if (!str_cmp(word, "PwdDigest")) {
        char* dig_text = fread_string(fp);
        decode_digest(ch->pcdata, dig_text);
        free_string(dig_text);
        fMatch = true;
        break;
    }
```

If this code is ever compiled in MSVC and run on an existing MUD, existing players will have their old, clear-text passwords read and converted to a digest. The old password is then thrown away, and never seen again.

This logic isn't possible on platforms where `crypt()` is available, unless I intended to keep the old encrypted password, and apply the new hash to _that_. But that case, I would have to keep `crypt()` around and use that "doubled digest" mechanism _everywhere_. I'm not interested in doing that. But if you wanted to, it should be trivial to do so.

If this is a newer player file, which the digest already persisted, I need only read the hex string, and run `decode_digest()` to get it back, exactly as it was when I first saved it.

And with that, player passwords are now safely hidden in plain sight. 

## Coda

This was the last "big" feature that I wanted in order to consider ROM Resurrected "complete". However, I've decided to go the extra mile and take on the most onerous task of this entire project: Ivan's OLC 2.1. I'll handle that next, and then call it done.

([Here is the code](https://github.com/bfelger/Mud98/tree/6bba8ccfb10aaa01a65f0023f8280526f611c4a4) with updates from this post.)

Copyright 2023, Brandon Felger