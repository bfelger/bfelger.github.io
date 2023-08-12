---
layout: default
title:  "Should I Convert My Ancient C Code to C++?"
date:   2023-08-08 20:00:00 -0500
categories: C C++ ROM MUD
---

# Should I Convert My Ancient C Code to C++?

The short answer is **no**.

The long answer is _maybe_.

The longer, longer answer is "that's the wrong question" (I'll explain).

This is an undertaking I've made many times over the years, many project, with varying results. And as someone who's been programming in both C and C++ for over 24 years, I consider myself enough of a grognard to have a rigid, dogmatic opinion of outsized self-importance on the issue.

Before porting ROM (or any legacy C project) to C++, you need to be able to answer these questions:
1. What am I hoping to gain by porting from C to C++?
2. Will I be able to write _correct_ C++ code?
3. What will I lose by porting to C++?

To answer these questions, it's important to consider the prospective language platforms.

## C vs C++

There is one thing I see in dev communities that always raises my hackles:

```
C/C++
```

This is an oxymoron. It's a contradiction in a single term. It's ridiculous.

_C/C++ does not exist._

There is modern, idiomatic C code. There is also modern, idiomatic C++ code. And _ne'er the twain shall meet_.

I have seen (and written, myself, in my younger, ignorant years) a sort of bastard "C with Classes" form of C++ that is simultaneously:

1. Terrible C code
2. Terrible C++ code

Now, it's true that C and C++ are deceptively similar in syntax, and yes, _most_ C code (but not all!) can be compiled by the C++ compiler. But these similarities are a trap for the unwary neophyte. In actuality, they are completely different languages.

To explain, here is what is (IMHO) the core _raison d'être_ of each language:

- **C** is WYSIWYG. It is predictable. It is transparent. You can tell what your resultant machine code is going to be by looking at the source code.
- **C++** is like most higher-level languages in that it is primarily about abstractions. But unlike most other such languages, it's about _zero-cost_ abstractions.

Idiomatic C code hides nothing. Idiomatic C++ hides as much complexity as possible. These are both good causes, but to opposite ends.

## Reasons Not to Port ROM to C++

There are many misconceptions C++ in particular, and OOP in general, that lead to misguided "upgrade" projects. But there are some cold, hard truths:

### OOP is a poor fit for ROM

> Note that I speak here not of OOP in the sense of a language construct, but as a comprehensive programming philosophy.

Why wouldn't ROM be perfect for Object-Oriented Programming? After all, everything in ROM is an object of some kind: a room, an area, a person, a thing. Each one is "tangible" in the context of game play. At first glance, it seems almost obvious that these things are objects (in the programming sense), and they should be defined by classes with control over the operations made on them.

But what about operations _between_ them? When you decide to toss an `OBJ_DATA` into a `ROOM_DATA`, who owns that operation? The Room, or the Object? This dilemma is satisfied most easily in pure-OOP languages like Java and C# by creating some kind of `RoomObjectManager` (implemented as a Singleton, naturally) to do pretty much what ROM already does in C: `obj_to_room()`.

In the end, ROM's tangible entities are not as important as the relationships between them and the actions performed to affect them.

> Check out ["Execution in the Kingdom of Nouns"](http://steve-yegge.blogspot.com/2006/03/execution-in-kingdom-of-nouns.html) by Steve Yegge. He is insightful enough that his prolific rants on a myriad of topics are often cited and quoted. He's also a MUD developer, to boot. He puts it better than I ever could.
>
> Please, consider it required reading. I was going to copy-and-paste a quote, but the whole thing is too dense with good insight to pick just one passage.

Now, C++ is an OOP-_enabled_ language, not an OOP-_enforced_ language. So you can port ROM and leave OOP to the wayside; and indeed, that is precisely what I would do if I did such a thing. C++ is, at heart, "mult-paradigm". It does not set out to enforce purity in any one programming philosophy.

In my option, good C++ code should do likewise.

### C++ is not a proper superset of C

The popular belief that C++ is "C plus other stuff" hasn't been true since C11. In fact, I've already introduced C11's zero-initializers into ROM Resurrected, which will not compile in C++ (not even, to the best of my knowledge, with language extensions).

```c
struct blah_t blah = { 0 };
```

C also has sparsely-designated aggregate initialization. In `const.c`, it would have made sense to do something like this:

 ```c
 const struct TableType table[MAX_ENTRY] = {
    [ELEM_1] = {"elem1", "thing1", ELEM_DATA_1},
    [ELEM_2] = {"elem2", "thing2", ELEM_DATA_2},
    [ELEM_3] = {"elem3", "thing3", ELEM_DATA_3},
    [ELEM_4] = {"elem4", "thing4", ELEM_DATA_4},
 //...
 ```
 ...where `ELEM_1` and so-on are macro definitions in `merc.h` This works even when the elements listed are non-continuous. This is not possible in C++, which does not permit this (then again, MSVC doesn't allow this in C, either, because their implementation of C99 is paltry).

This isn't just a matter of syntax, but also behavior. The C++ standard requires certain optimizations that are not required in C. Other popular C patterns, such as type-punning, are explicitly labeled UB ("Undefined Behavior") in C.

## Reasons to Add C++ to ROM

I've bashed on the prospect enough; now I'm going to list what are to _me_ compelling  reasons to add C++ into ROM.

Note that I did _not_ say "port". I said "add".

It is very much possible to place C and C++ in the same CMake project, and combine them seamlessly in the linker.  I'll talk a bout that more in bit.

### Resource Acquisition Is Initialization (RAII)

This, for me, is the #1 thing C++ brings to the table. ROM has tons of places that require bookend boilerplate for resource management.  Let's take, for instance, `BUFFER`, which I made use of in ROM Resurrected to avoid blowing the stack with huge `char` arrays on the stack:

```c
void foo(char* str) 
{
    BUFFER* buf = new_buf();
    add_buf(buf, str);
    //...Do lots of stuff
    free_buf(buf);
}
```

The first line creates a requirement for the last. This sort of "enforced manual boilerplate" is one of the things I will jump through hoops to avoid, and it's exactly what RAII intends to avoid.

So what would RAII look like, here?

Something like this:

```cpp
class Buffer {
public:
    Buffer()
    {
        val = new_buf();
    }

    ~Buffer()
    {
        free_buf(val);
    }

    void add(char* str)
    {
        add_buf(val, str)
    }

    BUFFER* val;
};
```

Now the `foo()` example can be written like so:

```cpp
void foo(char* str) 
{
    Buffer buf;
    buf.add(str);
    //...Do lots of stuff
}
```

RAII works via constructors and destructors on the resource object. When I instantiated `buf`, it implicitly called its default constructor (`Buffer()`), and implicitly called its default _destructor_ (`~Buffer()`) when it left scope. Because it was created on the stack, there is no `new` or `delete` keyword in sight. The code is svelte and the complexity is out-of-the-way.

> Note, however, that this is no longer the "C Way"; this is C++ code, and actually _bad_ C++ code, as I'm using a bare `char` array instead of a modern construct like `std::string` and `std::string_view`. 
>
> Like I said before: idiomatic, modern C++ looks _nothing_ like idiomatic, modern C.

### Exception handling

When I was first learning C++ back in '97, exceptions were heavy and slow. Many code guidelines banned them, most compilers have some option akin to `-fno-exceptions`, and many libraries overed "exceptionless" alternative builds.

However, the C++ Standard has come a long way, and so have its implementors. The goal of exceptions in C++ continues to be "zero-cost abstraction", and exceptions are an excellent tool to slay some of significant Gordian knots. 

As an example: if you read through "Resurrecting ROM", you may have seen my diatribe on `goto` and how willing to use it I am whenever I need to deal with multi-fail resource release. 

For instance:

```c
bool digest_message(const char *message, size_t message_len, unsigned char **digest, unsigned int *digest_len)
{
    EVP_MD_CTX *mdctx;

    if((mdctx = EVP_MD_CTX_new()) == NULL)
        goto digest_message_err;

    if(1 != EVP_DigestInit_ex(mdctx, EVP_sha256(), NULL))
        goto digest_message_err;

    if(1 != EVP_DigestUpdate(mdctx, message, message_len))
        goto digest_message_err;

    if((*digest = (unsigned char *)OPENSSL_malloc(EVP_MD_size(EVP_sha256()))) == NULL)
        goto digest_message_err;

    if(1 != EVP_DigestFinal_ex(mdctx, *digest, digest_len))
        goto digest_message_err;

    EVP_MD_CTX_free(mdctx);
    return true;

digest_message_err:
    // C++ exceptions are looking pretty nice, right now...
    ERR_print_errors_fp(stderr);
    EVP_MD_CTX_free(mdctx);
    return false;
}
```

> I am using this function solely to demonstrate how I can use exceptions in a logical manner; in reality, this is a perfect candidate for RAII, and I would actually prefer that over the example code below.
 
Which exceptions, this is easily rewritten:

```cpp
bool digest_message(const char *message, size_t message_len, unsigned char **digest, unsigned int *digest_len)
{
    EVP_MD_CTX *mdctx;
    bool rc = true;

    try {
    if((mdctx = EVP_MD_CTX_new()) == NULL)
        throw;

    if(1 != EVP_DigestInit_ex(mdctx, EVP_sha256(), NULL))
        throw;

    if(1 != EVP_DigestUpdate(mdctx, message, message_len))
        throw;

    if((*digest = (unsigned char *)OPENSSL_malloc(EVP_MD_size(EVP_sha256()))) == NULL)
        throw;

    if(1 != EVP_DigestFinal_ex(mdctx, *digest, digest_len))
        throw;
    } 
    catch(...) {
        ERR_print_errors_fp(stderr);
        rc = false;
    }

    EVP_MD_CTX_free(mdctx);
    return rc;
}
```

Now, this is a trivial implementation, and I would never use a default exception like that. But it demonstrates at least one way to use C++ to avoid a hated language construct like `goto`.

### Ownership and "move" semantics

The vast majority of run time crashes that I experienced in ROM development over the years is improper handling of memory. It was usually one of two things:
1. A reference to memory was copied, but the original reference remained, guaranteeing an eventual dereference of deleted data.
2. A reference to memory was forgotten, the memory tombstoned, and and memory began leaking.

ROM's game data is heavily relational, and these relations are maintained by a vast web of bilateral pointer references. An `OBJ_DATA`'s `carried_by` pointer refers to a `CHAR_DATA` that refers back to it in a linked-list of `OBJ_DATA`s in its own `carries` field  (via `OBJ_DATA`'s `next_content`). As you can imagine, modifying the placement of `OBJ_DATA`, whether in a room, a mob, or in another object requires extensive bookkeeping and pointer management.

C++ solved this issue with "smart pointers". This took some trial an error (like the misbegotten `std::auto_ptr`), but now modern C++ has settled on a preference for a clear division between _owning_ and _non-owning_ references instead of "raw" pointers.

- An _owning_ reference is explicitly in charge of the lifetime of the object. When it goes out of scope, so does the object. It can be copied, but it cannot be given away except via an explicit "move", which gives ownership to another reference (`std::move()`). This is implemented by means of `std::unique_ptr` which guarantees that there is only ever one `std::unique_ptr` to a given object. It cannot be passed except by const reference (there are exceptions to this rule, but they are largely common-sense and to your benefit).
- A _non-owning_ reference is one where the holder of the reference to an object has only transient access to it. It cannot make any decisions about its existence. It can't delete it, or replace it. Think something like `const Type&`, a non-null reference that can be interacted with, but not altered (except by means of its own, self-defined interface). 

Now, move semantics can be very annoying, as they are severely restricting on what you can do with owning references. But the restrictions are the feature: it is very difficult to inadvertently delete or lose such a reference.

### STL (specifically, containers)

Handling the grab-bag of objects that make up "held entities" in ROM is fairly annoying. Each thing (object, mob, person, etc.) has multiple linked lists specifying the "next" item in various linked lists. This makes extraction and placement operations complicated and difficult to change without introducing errors.

The STL (Standard Template Library) has a number of containers that can help abstract this away and leave us with a clean, isolated structure. Extraction and placement operations in this context deal only with the _container_, and not the other members of the container.

And while ROM uses linked-lists for all of this, it may not be necessary do so when using the STL. In fact, switching to a flat container (such as `std::vector`, which is the default and preferred "generic" container) can yield significant speed-of-access improvements due to _locality of reference_, something that is not really possible with linked-lists.

> "Locality of reference" is the proximity of repeatedly accessed items to each other in memory. When memory is accessed, it has to be fetched from RAM, and placed in the CPU on-die cache to be operated on. This can be a lengthy process. However, depending on how _often_ that area of memory is accessed, the CPU will keep it in place for as long as possible, which vastly reduces the amount of time it takes to fetch it again (or a nearby address).
>
> Arrays are the best structure to take advantage of locality of reference, especially when iterating over members serially. When the CPU goes to get the "next" item, the chances are _much_ higher that the item is already in the L1 cache, having already been fetched as part of the previous memory request. To make things easy on itself, the CPU never grabs just the memory you asked for; it grabs an entire "scan line" with surrounding pieces of memory. Using flat memory structures take advantage of this.
>
> Also, when "hot" areas of memory (the ones for which repeated requests are made) need to be swapped out of L1 cache, the CPU can choose to move it to a secondary on-die cache (L2), or the motherboard controller can keep it on a close-by tertiary cache. All of these options are much, much faster than a request to RAM "cold" storage.

## How Not to Use C++ in ROM

Here are things I know from experience, and many misbegotten attempts to "improve" ROM. 

### Compile it with a C compiler and call it "done"

Since _most_ C is also syntactically-valid C++, it is trivial to force C++ compilation for all C files.  Now you can use any C++ facility you want, including the wonderful examples listed above. The problem is that now you have a baseline of "C with classes" that is simultaneously terrible C _and_ C++ at the same time.

That isn't a problem, yet; but it _will_ be. _(stands on soapbox)_ The disparate philosophies of C and C++ have developed over almost _forty years_ (with C being over fifty). The ecosystems and knowledge bases formed around these tools are suited to these philosophies, and it is an error to mix them. _(gets off soapbox)_ From a more practical standpoint, integrating abstraction-driven code into WYSIWYG code will make it difficult to do either sufficiently to leverage their respective benefits, all the while introducing complexity that will make maintenance difficult.

### Replace `char*` with `std::string`

Back in '98, when GCC was maintained by a single person, there was a new project at GNU called `egcs` that was intended to create new, scratch-written C and C++ compilers intended to integrate upcoming standards changes for both platforms. We (the implementors for our MUD, Cities of M'dhoria) wanted take advantage of this newer compiler, so we made the switch from `gcc` to `egcs`. Since we got C++ for free out-of-the-box, why not use it?

> Fun fact: just a few years later, `egcs` was renamed to GCC, and became its official incarnation. The old codebase was abandoned.

Because I was in CS 201 "Introduction to Computer Programming", I decided to do away with `char*` and switch it all to `string` (not `std::string`, mind you; my CS professors insisted we put `using namespace std;` at the top of every code file. _Yecchh!!!_).

The result: the memory footprint _did_ improve (spuriously, as I'll explain) , but we still had to make frequent conversions to and from `char*`, and speed of execution fell dramatically. We never solved that issue (with full-time school and work, I frequently fell off the face of the planet [Sorry, Jeremy!]), but I am sure I know how to solve the issue, now (more on that, later).

Now, what happened? The best way to explain is to couple it with the next terrible way to use C++ in ROM:

### Replace all the linked-lists with STL containers

Wait... didn't I just sing paeans to the glory of STL containers, and talk about how wonderful the locality of reference of `std::vector` is?

There's one hitch, here, and it's a big one. It's the very same reason why performance fell when Cities of M'dhoria switched from `char*` to `std::string`: memory management.

Whoever wrote ROM's memory allocation scheme was _very_ clever. It is very fast, and it's pretty hard to beat. The STL, however, uses `std::allocator`, a general-purpose allocator which, in most implementations, calls `malloc()` under the covers. This makes using STL containers (of which `std::string` is one; specifically, it's a `std::vector<char>` [a gross over-simplification, but essentially correct]) much slower that ROM's own `alloc_mem()`, which uses very fast recycle buckets (but with some size bloat, which is way `std::string` had an edge on memory footprint).

Solving this problem is very possible, but it requires delving deep into the black magic of C++, far beyond what the everyday programmer will do: writing a new allocator and assigning it to an assortment of custom STL containers.

That isn't something a C++ neophyte should be expected to do easily, but it is a very common problem in video game development; such that there are now off-the-shelf solutions. However, with resources like `std::polymorphic_allocator` in the STL, they may no longer be necessary. Myself, I prefer to write my own allocators (they can even use `alloc_mem()` behind the scenes; this is a boon, as then there aren't two different, disparate memory management schemes in the project).

### Put everything in a class

This is, by far and away, the _worst_, most _evil_ thing that modern OOP-enforced languages have done to us. It is now the default expectation that all functions are tied to a specific set of data.

Take a lesson from the STL: all functions are tied to a specific _namespace_, not a specific owning class. But what's worse is the fact that for all that, both Java and C# are littered with bare classes with `static` methods just to get to the same functionality as C++'s free functions.

That, in my mind, is proof that these are clear language design failures, and I believe the proof is in the pudding: Kotlin, Java's upstart usurper, has free functions, and C# has them now, as well.

Classes own data. But _actions_ belong to _functions_. Class methods should facilitate data access, and otherwise contain only operations that are logically self-contained and isolated to the data of the class itself. Nothing else belongs.

Okay, enough about what _not_ to do. 

## How to Use C++ in ROM

Fair warning: this is a _highly opinionated_ section where I speak from a position of overly-inflated self-importance and intellectual certitude.

That's because I'm right.

I'm _always_ right.

>  _...is my wife anywhere nearby to see this?_

Anyway...

At the beginning, I said that asking _should I port C++ to ROM?_ was the "wrong question." The right question is this: _how can I leverage C++ in my existing C project?_

### Keep C++ code isolated with an `extern "C"` interop

As previously mentioned, a common first step of mixing C and C++ in a single project is the issue of _name-mangling_. Because C++ supports _overloading_ (where a function name can be reused for different functions [e.g. for different arguments, so like `send_to_room(OBJ_DATA*, ROOM_DATA*)` and `send_to_room(CHAR_DATA*, ROOM_DATA*)`]), different namespaces, as well as member functions of different classes, the compiler uses name-mangling to disambiguate. In the previous example, the two different versions of `send_to_room()` would actually compile to `_Z12send_to_roomP8obj_dataP9room_data` and `_Z12send_to_roomP9char_dataP9room_data` in MSVC, respectively.

Now, if we're going to add C++ code, we _want_ this facility, but C code cannot call name-mangled C++ functions. The solution is to create a separation layer between C and C++ using de-mangled functions to access C++ code from C. It just has a few rules:
- They must be top-level, global-scope identifiers.
- They must be declared with `extern "C"`.
- They must hide name-mangled code (like classes and their methods, anything nested, and overloaded functions).
- They must overwise follow all the rules of C naming conventions (e.g. no overloading).

Here's an example. Let's say I wanted to introduce RAII to my SHA256 hash function, as previously mentioned, and went with a solution like this, in `digest.h` and `digest.cpp`, respectively:

```cpp
class Digest {
public:
    Digest();
    ~Digest();

    int update(char* message, size_t message_len);
    int finalize(unsigned char **digest, unsigned int *digest_len);

private:
    EVP_MD_CTX *mdctx;
};
```
```cpp
Digest::Digest()
{
    // FOR DEMONSTRATION ONLY!
    // I would spend more than 5 minutes crafting a solution that doesn't
    // throw exceptions from the constructor.
    if((mdctx = EVP_MD_CTX_new()) == NULL)
        throw;

    if(1 != EVP_DigestInit_ex(mdctx, EVP_sha256(), NULL))
        throw;
}

Digest::~Digest()
{
    EVP_MD_CTX_free(mdctx);
}

void Digest::update(char* message, size_t message_len)
{
    if(1 != EVP_DigestUpdate(mdctx, message, message_len))
        throw;
}

int Digest::finalize(unsigned char **digest, unsigned int *digest_len)
{
    // Again, demonstration only.
    // I would use a separate UDT for handling the result using RAII.
    if(1 != EVP_DigestFinal_ex(mdctx, *digest, digest_len))
        throw;
    } 
}
```

This is a hash solution that implements RAII; simply declaring an instance of `Digest` ensures that its EVP context is cleaned up when it leaves scope without us having to remember to do so.

However, C can't use this class (and wouldn't know how to use its constructors or destructors even if it _could_).

Here's an interop solution for `digest.h` and `digest.cpp`, respectively: 

```cpp
#ifdef __cplusplus 

class Digest {
    // blah blah blah
};

extern "C" {
#endif

bool digest_message(const char *message, size_t message_len, 
    unsigned char **digest, unsigned int *digest_len)

#ifdef __cplusplus
}
#endif
```

```cpp
extern "C"
bool digest_message(const char *message, size_t message_len, 
    unsigned char **digest, unsigned int *digest_len)
{
    bool rc = true;

    try {
        Digest d;
        d.update(message, len);
        d.finalize(digest, digest_len);
    } 
    catch(...) {
        ERR_print_errors_fp(stderr);
        rc = false;
    }

    return rc;
}
```

It doesn't quite approach what I would call "quality" code, but it gets the point across. In this case, I created a "shim" function that bridges C and C++ code, and can be called from either. 

> I have a convention that I am fairly certain is not in popular use, but it helps more clearly define the C-to-C++ DMZ. I use `.h` as the extension for C code, and `.hpp` for C++. Popularly, `.h` is used for both (or, like the STL does, the extension is omitted altogether).
>
> The immediate convenience of this is that I never have to worry about accidentally bringing in mangled identifiers to my C code: I simply have to remember not to include a `.hpp` header from a `.c` source file.

Any C header that is included by C++ code needs to have an `extern "C"` guard:

```cpp
#ifdef __cplusplus
extern "C" {
#endif

// Stuff

#ifdef __cplusplus
}
#endif
```

In fact, I would just go ahead and put it in _all_ C headers.

> Since I use `.hpp` for pure-C++ headers that are never included by C code, I omit this guard for those headers.
> 
> Also, some folks do this in their `.cpp` files:
> ```cpp
> extern "C" {
>     #include "c_only_header.h"
> }
> ```
> Don't do that.

By now, it should be obvious that the original, overloaded example of `send_to_room()` can't go in an `extern "C"` context, because they would have the same demangled symbol. But using a shim function in the interop layer makes it trivial:

```cpp
extern "C"
void send_obj_to_room(OBJ_DATA* obj, ROOM_DATA* room)
{
    send_to_room(obj, room);
}

extern "C"
void send_char_to_room(CHAR_DATA* ch, ROOM_DATA* room)
{
    send_to_room(ch, room);
}
```

But at this point, have you actually saved anything by overload? _I_ don't think so. Not for this example, anyway. But if you are being a good C++ developer, and putting your C++ functions in a proper namespace, then the above code become _essential_ (and it will be named `rom::send_to_room()` or some such).

### Take it a chunk at a time

Taking a single piece of ROM and putting it in C++, and writing all the functions needed to be called from C to utilize it will start out laborious. Eventually, however, the pieces that are ported will more easily fit with the existing C++ code than it did with the remaining C code it left behind.

Doing the granular approach also gives you the opportunity to enforce (and adhere to) Single-Responsibility Principle. Strong demarcations in code functionality ease the cost of maintenance, increase code quality, and mitigate the risk of making changes.

### Don't use OOP

I do _not_ mean "don't use classes", but rather the broader philosophy of OOP, as expressed by its four tenets:
- Abstraction
- Encapsulation
- Inheritance
- Polymorphism

Now the first two are _fine_. In fact, without them there is really no point to C++, at all. But the latter two have proven problematic _when applied without consideration_.

Here's a case study that is totally made up, hypothetical, and not something I actually wasted dozens of hours on in my youth. _I swear_.

Let's say, one day you notice that everything in ROM seems to have a "name", a "description", a "short description", a "long description", and an "extra description". So you make a `Description` base class. And then all of these can inherit from that, right?

```cpp
class RoomData : public Description { /*...*/ };
class CharData : public Description { /*...*/ };
class ObjData: public Description { /*...*/ };
```

Hey, that's handy. Now each of these classes is _also_ a `Description`. Thanks to polymorphism, we can use these wherever a reference to `Description` is called for (but it has to be a reference! Otherwise it's _slicing_, and it's a problem that only exists thanks to polymorphism).

> ROM uses `ANGRY_SNAKE_CASE` for ADT's, but I prefer `PascalCase`. I'm obviously foreshadowing what I'm going to do with ROM Resurrected, eventually.

Now, we further notice that `CharData` and `ObjData` share "material" and "affects", and both are placable together in a room:

```cpp
class RoomData : public Description { /*...*/ };
class CharData : public Description, public Tangible { /*...*/ };
class ObjData: public Description, public Tangible { /*...*/ };
```

That, right there, is called _multiple inheritance_, and it's problematic (it's not _evil_, like many modern OOP devotees think, but it's solidly problematic).

Here's how it can rear its ugly head:

Let's say, given the above, that I decided to add a vehicles to ROM. It's an object, but one you can climb inside, like a room. Here's the naive implementation:

```cpp
class VehicleData : public RoomData, public ObjData { /*...*/ };
```

In OOP, this is called the "diamond of death":

```
         VehicleData
          /      \
     RoomData   ObjData
          \      /
         Description
```

This implementation of `VehicleData` includes (in a publicly resolvable fashion) the attributes of both `RoomData` and `ObjData`, both of which _also_ bring along `Description`. Any attempt to refer to `VehicleData`'s `Description` components yields a compiler error, as they are now ambiguous. There are many strategies to disambiguate (because we've had over forty years to solve this problem), but we shouldn't really _have_ to. I don't want to complicate my code to support OOP this way.

The problems that were once solved reflexively with inheritance and polymorphism are often (not always, but often) better handled by other means. This has been known at least since the release of "Design Patterns: Elements of Reusable Object-Oriented Software" by the Gang of Four* in 1995.

> \*Not to be confused with the Gang of _Three_ of "Dragon Book" fame.

This book doesn't _subvert_ OOP, so much as it provides an alternate, better way to _implement_ OOP. It has a couple tidbits of wisdom, but the ones that have had the biggest impact are these:
- Program to an interface, not an implementation.
- Favor object composition over class inheritance.

_"Program to an interface, not an implementation"_ means this: focus on external behaviors, not internal attributes. In the above example, `VehicleData` tries to combine all the internals of `RoomData` and `ObjData`; that is, it is coded against their _implementation_.

In modern OOP, we would describe _interfaces_ to describe the expected behavior. These interfaces don't have any code; all they do is establish a _promise_ to observers that each of these classes fulfill certain behaviors.

In C++, interfaces are implemented as _abstract classes_, which contain only `virtual` "deleted" functions, like so:

```cpp
class InterfaceExample {
    virtual const char* getter_func() = 0;
    virtual void setter_func(const char*) = 0;
    // ...
};
```

You can't create or initialize an interface; you can only add it to another class as a promise of fulfillment.

In the previous example, I would make a `Describable` interface that promises that objects that type will have functions that expose the description elements (and throws a compiler error if it doesn't), and a... I dunnow... `Enterable` interface that promises functions to add and remove people and objects, and establish exits to/from other `Enterable`s (yeesh, I _hate_ that name, now). `Tangible` already sounds pretty good as an interface for physical things.

```cpp
class ObjData: public Describable, public Tangible { /*...*/ };
class RoomData : public Describable, public Enterable { /*...*/ };
class VehicleData : public Describable, public Tangible, public Enterable { /*...*/ };
```

_Favor object composition over class inheritance_ means this: don't throw a base class's constituent parts into a subclass; rather, the latter should keep a reference to the former as a concrete object. This establishes a "has a" relationship, rather than a "is a" one.

For instance, in the above example, we could have a `Description` concrete class, like before, that `ObjData` contains an instance of as a member. Its implementation of `Describable` would then be "passthrough" functions that act on the hidden `Description` member, like so:

```cpp
class ObjData : Describable {
public:
    const char* short_desc() override 
    { 
        return desc.short_desc;
    }

    void short_desc(const char* sd) override
    {
        desc.short_desc = sd;
    }

private:
    Description desc;
};
```

This example uses both interfaces _and_ composition.

> The title of this section is "Don't use OOP." But I would like to amend it: "Use modern OOP so as to avoid the many pitfalls that have befallen your elderly and befuddled forerunners".

### Use only one memory allocation scheme

Until you know what that is, I would just use `alloc_mem()`. You can make the STL use this scheme for ROM objects by means of a custom implementation of `std::allocator_traits`. Alternatively, you can write a custom `std::allocator` and pass that as a template parameter to customized, aliased STL containers. That's pretty deep in the hole, though, as far as the black magic of C++ goes. 

As a first step in introducing C++ to ROM, I recommend against turning any existing C structs into C++ classes (or porting those structs without `extern "C"`, for that matter). But if you find your self writing new classes entirely for the C++ side of things, and you want to use the ROM allocation scheme, then you would do something like this _(requires C++20)_:

```cpp
#include <memory>

// Initialize and construct
Blah* obj = construct_at(reinterpret_cast<Blah*>(alloc_mem(sizeof(Blah))));

// Destroy free free
std::destroy_at(obj);
free_mem(obj);
```

## Final Thoughts

As you can see, properly integrating C++ into ROM is a _vast_ undertaking, but one that can have enormous benefits (and attendant pitfalls). But keep this in mind: C remains, as ever, a fantastic language that is more than capable of sustaining a project like ROM for many more decades to come.

If you choose to go the route of C++, make sure you are doing it for the right reasons, at the right time, and _in the right way_.

Copyright 2023, Brandon Felger
