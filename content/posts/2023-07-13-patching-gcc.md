---
categories: c
date: "2023-07-13T00:00:00Z"
slug: patching-gcc-cosmo
title: Patching GCC to build Actually Portable Executables
---

**2023-07-13:** I wrote a [\~2000-line `gcc` patch][patch] to simplify building
Actually Portable Executables with [Cosmopolitan Libc][cosmo]. Now you can build
popular software such as `bash`, `curl`, `git`, `ninja`, and even `gcc` itself,
with Cosmopolitan Libc via the `./configure` or `cmake` build system, without
having to change source code, and the built executables should run on Linux,
FreeBSD, MacOS, OpenBSD, NetBSD, and in some cases even Windows too[^windowspls].
Here's how you can port your own software to Cosmopolitan Libc now:

1. Clone the [Cosmopolitan Libc repo][cosmo], and set up the `/opt/cosmo` and
   `/opt/cosmos` directories as per the README.
2. Set the necessary environment variables:

```sh
export COSMO=/opt/cosmo
export COSMOS=/opt/cosmos
export CC=$COSMO/tool/scripts/cosmocc 
export CXX=$COSMO/tool/scripts/cosmoc++
export LD=$COSMO/tool/scripts/cosmoc++
```

3. _(Optional)_: here are my forks of [`gcc`][patch] and
   [`musl-cross-make`][buildpatch], which you can use to build `gcc` with the
   latest version of my patch. If you do this, remember to edit `cosmocc` and
   `cosmoc++` accordingly.
4. Let's try a simple `hello world` example first:

```c
#include <stdio.h>

int main() {
    printf("Actually Portable Executable built via cosmocc\n");
    return 0;
}
```

To build the APE for a single file, you can just

```sh
cosmocc -o hello.com hello.c
```

5. to build software that [uses `./configure`][superconf] or `cmake`, export the
above environment variables, and then run the necessary build steps to get a
static debug executable that runs on Linux

6. to get an Actually Portable Executable, run 

```sh
objcopy -SO binary your-software your-software.com
$COSMO/o/tool/scripts/zipcopy.com your-software your-software.com
```

When Cosmopolitan Libc burst onto the scene with the [Hacker News post about the
`redbean` webserver][redbeanhn] in 2021, an immediate question was how existing
C software would run on it. The best way to test a new libc is to build more
code on top, and [many codebases have been ported][thirdparty] to use cosmo. The
porting efforts have also helped fill out the libc API. However, a new libc must
also fit in with how C software is conventionally built -- things like
`./configure` and `cmake` -- so that porting more software is easy. When
[`pthreads` became available last year][pthreads], I said: [porting is just a
matter of convincing the build system][rustpost].  I believe my `gcc` patch is a
big step towards seamlessly building a lot of software with Cosmopolitan Libc.
This blog post will cover how the patch came to be.

## Introduction

[Lua][lua] was the first programming language to be ported to Cosmopolitan Libc,
followed by [Wren][wren], [Janet][janet], and Fabrice Bellard's
[`quickjs`][quickjs]. There appeared to be a common change across the ports:
`switch` statements with system values like `SIGTERM` or `EINVAL` had to be
rewritten as `if` statements.  **2021-03-24:** I raised issue \#134 on Github
about this, and got the below answer from [Justine Tunney][jart]:

> Code that uses `switch` on system constants needs to be updated to use `if`
statements. That's unavoidable unfortunately. It's the one major breakage we
needed to make in terms of compatibility. The tradeoff is explained in the APE
blog post [...] C preprocessor macros relating to system interfaces need to be
symbolic. This is barely an issue, except in cases like `switch(errno){case
EINVAL:}` If we feel comfortable bending the rules [...]

**2022-09-05:** After writing a bunch of ports, it was clear that the
`switch(errno)` pattern is quite common, and porting software would be much
faster if the rules could be bent automatically.  I decided to try solving
`this`, but let's define what `this` is.

## `switch` to `if`

Consider the below code snippet:

```c
// somewhere within your code
switch (errno) {
    case EINVAL:
        printf("you got EINVAL\n");
        break;
#ifdef EAFNOSUPPORT
    case EAFNOSUPPORT:
        // fallthrough
#endif
    case ENOSYS:
        printf("you got ENOSYS\n");
        break;
    default:
        printf("unknown error\n");
        break;
}
```

The C standard[^cstandard] says that `case` labels need to be compile-time
constants.  If `EINVAL` is not a compile-time constant, you would get an error
like `case label does not reduce to an integer constant` when compiling this
snippet with `gcc`.  So you'd need to rewrite the `switch` statement into an `if`
statement. My code rewrites `switch` statements like this:

```c
// rewriting the switch to an if
{ // maintain scoping
    if(errno == EINVAL) goto caselabel_EINVAL;
#ifdef EAFNOSUPPORT
    if(errno == EAFNOSUPPORT) goto caselabel_EAFNOSUPPORT;
#endif
    if(errno == ENOSYS) goto caselabel_ENOSYS;
    goto caselabel_default;
    caselabel_EINVAL:
        printf("you got EINVAL\n");
        goto endofthis_switch; //  break
#ifdef EAFNOSUPPORT
    caselabel_EAFNOSUPPORT:
        // fallthrough
#endif
    caselabel_ENOSYS:
        printf("you got ENOSYS\n");
        goto endofthis_switch;
    caselabel_default:
        printf("unknown error\n");
        goto endofthis_switch;
    endofthis_switch:
        ((void)0);
}
```

The above pattern might not be the most elegant[^harmgoto] but it handles all
the examples I have seen across codebases.

## `struct` initializations

While constructing test cases for `switch` statements that would have to be
rewritten, I came across a related problem that also assumed compile-time
constants -- `static` or `const` `struct` initializations. The [`faulthandler`
module in CPython][faulthandler] is a real-life example of this, but let's look
at the below snippet:

```c
struct toy {
    int status;
    int value;
};

void func() {
    struct toy t1 = 
        {.status = EINVAL, .value = 25}; // ok
    const struct toy t2 = 
        {.status = EINVAL, .value = 25}; // error
    static struct toy t3 =
        {.status = EINVAL, .value = 25}; // error
}

const struct toy gt2 = 
    {.status = EINVAL, .value = 25}; // error
static struct toy gt3 =
    {.status = EINVAL, .value = 25}; // error
```

If `EINVAL` is not a compile-time constant, four of the initializations above
are not valid[^cppdiff] in C. My fix was to dummy-initialize the `struct`s and
then add an `if` statement or an [`__attribute__((constructor))`][ctor] to fill
in the correct value(s) before they are used at runtime.


## Outline of the problem

From the above two sections we have defined the problem space: certain `switch`
statements and `struct` initializations may not compile, because they rely on
system values being compile-time constants. Now:

* Can we (automatically) find these `switch` statements and convert them into
  `if`s and `goto`s?
* Can we (automatically) find these `struct` initializations and insert the
  necessary code that fills in the correct runtime values?

I started off trying to automate these conversions using `sed` in a shell
script.  After a while, it was a python script with some extra regex work to
handle `switch` fallthroughs. But neither of these worked completely because of
the C preprocessor and `ifdef`s. It is difficult to perform the rewrite as a
_text substitution_ when you do not know which `ifdef`s would activate during
compilation. Then, maybe I could perform the rewrite as an _AST substitution_ --
if the code was available as an AST (abstract syntax tree), performing a rewrite
would be replacing one subtree with another. I could also avoid dealing with
the C preprocessor...

## Curiously exploring `gcc` plugins

`gcc` provides a plugin architecture via which you can access the AST of the
code being compiled. You compile your code as a shared object, load it alongside
`gcc` with the flag `-fplugin=your-plugin.so`.  The [`gcc` internals
documentation][gccint] provides detailed explanations as to when and how the
plugin writers can interact with the compilation process, and there are some
[wonderful][ex1] [articles][ex2] with examples of how you can write your own
`gcc` plugins (perhaps I should write my own article with examples). For now,
let's just remember the following:


1. `gcc` allows plugins to activate callbacks during [plugin events][pluginevent]:

```c
enum plugin_event
{
  PLUGIN_START_PARSE_FUNCTION,  /* Called before parsing the body of a function. */
  PLUGIN_FINISH_PARSE_FUNCTION, /* After finishing parsing a function. */
  PLUGIN_FINISH_DECL,           /* After finishing parsing a declaration. */
  PLUGIN_PRE_GENERICIZE,        /* Allows to see low level AST in C and C++ frontends.  */
  PLUGIN_INFO,                  /* Information about the plugin. */
  PLUGIN_INCLUDE_FILE,          /* Called when a file is #include-d */
  /* [removed some of the events] */
  PLUGIN_START_UNIT,            /* Called before processing a translation unit.  */
  PLUGIN_FINISH_UNIT,           /* Useful for summary processing.  */
  PLUGIN_FINISH                 /* Called before GCC exits.  */
};
```

The callback is provided some data from `gcc` (which may be a pointer to an AST,
a string, or something else depending on the event), and a pointer to data that
you might have initialized at startup. Since we need to manipulate the AST, the
`PLUGIN_PRE_GENERICIZE`, `PLUGIN_FINISH_PARSE_FUNCTION`, or `PLUGIN_FINISH_DECL`
events seem viable. 

2. The AST provided by `gcc` is a tree structure, which takes a while to connect
   to the original C code, but is easy to read and manipulate afterwards. `gcc`
   provides a function called `debug_tree` that prints the AST.  It's probably
   the function you will use the most when developing a plugin.  There are also
   convenient macros to access, say, the second arg of the `ADD_EXPR` or the
   name of the `VAR_DECL`.

3. `gcc` provides a function `walk_tree_without_duplicates(ast, your_callback,
   your_data)` which walks through the entire AST in pre-order traversal and
   presents each node to `your_callback` to read, process, and modify.

**2022-12-27:** I set up a [`gcc` plugin][oldplugin] that would activate on
`PLUGIN_PRE_GENERICIZE` and walk through the AST looking for `switch` statements
to rewrite. Simple enough, seemed like it would work, so I tried the below
example:

```c
switch(errval) {
    case 0:
        break;
    case SIGILL:
        printf("you got a SIGILL\n");
        break;
    default:
        printf("unknown error\n");
}
```

But:

```gcc
examples/ex1_modded.c: In function ‘exam_func’:
examples/ex1_modded.c:25:5: 
   error: case label does not reduce to an integer constant
   25 |     case SIGILL:
      |     ^~~~
```

`gcc` is still raising an error due to the `case` label? Why does the plugin not
activate? I check the AST with `debug_tree`, and `case 0:` was there:

```
<switch_stmt
   arg:0 <parm_decl errval>
   arg:1 <statement_list type <void_type void>
     stmt <case_label_expr type <void_type void>
         arg:0 <integer_cst constant 0> arg:2 <label_decl D.2390>>
     stmt <break_stmt type <void_type void>>
```

But `case SIGILL:`?

```html
     stmt <call_expr type <integer_type int>
         side-effects
         fn <addr_expr type <pointer_type>
             constant arg:0 <function_decl printf>>
         arg:0 <nop_expr type <pointer_type>
           arg:0 <addr_expr type <pointer_type>
             arg:0 <string_cst type <array_type>
                readonly constant static "you got a SIGILL\012\000">>>
     stmt <break_stmt type <void_type void>>
```
 
There was no `case` label to rewrite! I checked my code and all of the build
config for building the plugin, but I always got an invalid AST, and a `switch`
statement without the (incorrect) `case` label that I wanted to rewrite. 

After a while watching the errors repeat, I found out that the plugin can access
the AST only _after_ `gcc` had finished parsing the source file, and the `case`
label validation happens _during_ the parsing process. I can't fix the `switch`
case without knowing the value of the `case` label, and even then, the error
raised by `gcc` earlier would still mean the compilation fails.

Was this the end?

## Return of the C preprocessor

**2023-03-14:** `PLUGIN_PRE_GENERICIZE` only received an AST after parsing, and
but `gcc` gave me an invalid AST due to the `switch` statement. I needed to
ensure a valid AST before I could rewrite it, and one way to interact with the
code before parsing is ... with the C preprocessor! Could I interact with the C
preprocessor from within the plugin? Yes! The `gcc` plugin headers provide a
structure called `cpp_reader`, within which you can define custom callbacks that
activate when a macro is `#define`'d, `#undef`'d, and used! I still don't
understand exactly how `cpp_reader` works, but now there was another line of
attack:

* Cosmopolitan Libc (used to) provide macros for system values like below, so
  that the `#ifdef` checks would not complain:

```c
extern const int SIGTERM;
#define SYMBOLIC(X) X
#define SIGTERM SYMBOLIC(SIGTERM)
```

* With a custom header file, I could create a temporary value like:

```c
static const int __tmpcosmo_SIGTERM = 3141592;
```

* and then within my plugin code, I could _intercept the usage of the macro
  `SYMBOLIC(SIGTERM)` and substitute my temporary value instead_, so that the
  parsing would not error out.

* Finally, with a valid AST in `PLUGIN_PRE_GENERICIZE`, I could _look up the
  specific line of code where I did the interception_ and rewrite the AST with
  the correct `VAR_DECL` of `SIGTERM` instead of the temporary value.

Let me repeat: a `#define` in Cosmopolitan Libc headers, a temporary `static
const` in my custom header, a macro interception in the plugin, followed by an
AST rewrite in the plugin. In terms of code outside the compiler:

```c
// Cosmopolitan Libc provided
extern const int ENOSYS;
#define ENOSYS SYMBOLIC(ENOSYS)
// for the plugin-macro-hack
static const int __tmpcosmo_ENOSYS = 1209372;
#define SYMBOLIC(ENOSYS) __tmpcosmo_ENOSYS
```

Then the plugin would latch on to the temporary constants, and perform the AST
rewrites. It sounds ridiculous, [but it _worked_][pluginhere].  Within that
week, I had a plugin that successfully worked around both the `switch`-`case`
and the `struct` initialization errors, which I could verify with a bunch of
simple examples. **2023-03-26:** I got a minimal [CPython 3.11][tempcpy311]
building with this macro hack.  There were a few issues, but at this point the
experiment had a decent chance of success, so I asked Justine if we could use
this plugin when building software with Cosmopolitan Libc.

## Why not just patch `gcc`?

The common theme in my discussion with Justine was: now that we know the problem
can be solved, is there a simpler way to solve the problem? She suggested I not
bother using plugin APIs, work on the GCC codebase itself, and simply change
whatever I needed. I was initially hesitant[^readgcc], but the edge cases and
crashes with the macro-hack arrangement got wackier[^dblseg] , due to lots of
extra unnecessary work. Here's an example:

```c
extern const int ENOSYS;
/* for the plugin */
static const int __tmpcosmo_ENOSYS = 172389;
#define SYMBOLIC(ENOSYS) __tmpcosmo_ENOSYS
#define ENOSYS SYMBOLIC(ENOSYS)

int foo() {
    switch(errno) {
        case ENOSYS:                        // necessary to rewrite
            int x = ENOSYS + 1;             // EXTRA WORK
            printf("ENOSYS = %d", ENOSYS);  // EXTRA WORK
            return x;                       // EXTRA WORK
        default:
            return 0;
    }
}
```

The only error that `gcc` would raise without the plugin is the `case ENOSYS`,
and the plugin can handle that. However, the macro substitution due to
`ACTUALLY(ENOSYS)` meant that the plugin also had to handle every other (valid)
use of `ENOSYS`, which could (hopefully) be avoided by patching `gcc` instead. 

It took a while to figure out where to add my code into the `gcc` source tree,
but the overall design of the plugin became simpler.  Instead of `-fplugin`, the
code was now activated with a compiler flag `-fportcosmo`.  To my great relief,
I could delete all my macro-related hacks. Now I could just intercept the parser
error instead: right before `gcc` raised a `case is not constant` error, check
if `flag_portcosmo` is active, and if yes, perform the necessary substitution.
The AST rewriting code was a copy-paste from the plugin, and it was executed
by `gcc` right before invoking other plugin callbacks.  

## Porting software with the patched `gcc`

**2023-06-05:** At this stage, the patched `gcc` was passing all the test cases
I had written for the plugin earlier, and new binaries were added into the
Cosmopolitan Libc monorepo.  This allowed for compiling a lot more code, leading
to fixes and improvements. Building `lua` was now straightforward, and for
`python3.11` I added some changes to access the ZIP store within the executable.
I found out where `g++` raised the `case constant` error so `ninja` could build.
Then I tried building `busybox`, but my brain had a segfault with the [below
code][busyboxpls]:

```c
static const char signals[][7] ALIGN1 = {
	[0] = "EXIT",
#ifdef SIGHUP
	[SIGHUP   ] = "HUP",
#endif
#ifdef SIGINT
	[SIGINT   ] = "INT",
#endif
#ifdef SIGQUIT
	[SIGQUIT  ] = "QUIT",
#endif
#ifdef SIGILL
	[SIGILL   ] = "ILL",
#endif
#ifdef SIGTRAP
	[SIGTRAP  ] = "TRAP",
#endif
```

That's an array of strings, _index-initialized by the signal constants_, used as
a lookup table, just to convert signal values to strings and back.  <strike>Why?
Why not just use a switch statement? What if one of the constants turns out to
be 400000, does <code>gcc</code> allocate a large array with empty
space?</strike> Yeah... my patch can't handle that, and I don't want it to.
Anyway, I wanted [Python3.11][cpy311] to have `ncurses`, so I tried building
that next.  `ncurses` had [something interesting][ncurseswow]:

```c
typedef struct {
    unsigned int val;
    const char name[BITNAMELEN];
} BITNAMES;

#define DATA(name)        { name, { #name } }
#define DATA2(name,name2) { name, { #name2 } }

static const BITNAMES cflags[] =
    {
	DATA(CLOCAL),
	DATA(CREAD),
	DATA(CSTOPB),
#if !defined(CS5) || !defined(CS8)
	DATA(CSIZE),
#endif
	DATA(HUPCL),
	DATA(PARENB),
	DATA2(PARODD | PARENB, PARODD),
	DATAX()
#define ALLCTRL	(CLOCAL|CREAD|CSIZE|CSTOPB|HUPCL|PARENB|PARODD)
    };
```

Now `PARENB` and `PARODD` were `extern const` values in
Cosmopolitan Libc, and my patch was fixing the `struct` initialization for
`DATA(PARENB)` as expected, but it was getting stuck with:

```c
// DATA2(PARODD | PARENB, PARODD) evaluates to
 {.val = PARODD | PARENB, .name = "PARODD"}, // unable to rewrite
```

The `struct` initialization for `PARODD` had a _binary expression_, instead of
the usual constants, and my patch had no code to handle that. I thought about it
for a while, and decided situations like this would occur frequently enough
where it would be convenient to handle them automatically. I added a change
allowing `case` labels and `struct` initializer elements in C[^notcxx] to be
_arbitrary expressions_, which means you can compile things like:

```c
 // C does NOT allow this, but with -fportcosmo...
case (foo(SIGTERM) + bar(SIGILL)):
    break;

struct toy t = 
    {.status = (foo(SIGILL) + bar(ENOSYS)), value = 22};
```

With that, `ncurses` built without a complaint. Note that the C standard allows
`case` labels to only be compile-time constants, and my patched compiler will
still raise a warning when you do something like this, but otherwise...

![](/images/patch-gcc/portal_gun.webp)
<p class="caption"> <code>./configure</code> flags, run <code>make</code> and ... you got it! you f****** got it!</p>

## Closing Notes

**2023-07-13:** To port software to Cosmopolitan Libc, you just need to convince
the build system. I mentioned this last year, and this `gcc` patch experiment
was to reduce the amount of convincing you would need to do. One of the
motivations for this experiment was to find an answer to the question "what is
the minimum amount of source code that would need to change in order to port
something to Cosmopolitan Libc?" I am glad to find out that the answer is less
than ten lines in most cases, and could even be zero.

Of course, my patch isn't perfect. It can't handle some anonymous structs,
`enum`s, `const int`s, or amazing things like using `SIGILL` as an array index
within an initializer.  Rare compiler crashes may still occur in some weird
`static` initializations, or if you try stuffing `SIGTERM` into a `static const
int8_t`. But I've spent a good chunk of time removing obvious counterexamples,
and a lot of popular software builds seamlessly. A stringent testing setup will
reveal more things to improve.

If you prefer, you can still rewrite the `switch` statements and `struct`
initializers by hand, but for many cases the compiler can do it for you, and all
that is necessary for a port is to specify the right flags to `./configure` or
`cmake`. If you can build your C software statically (bonus points if it builds
with `musl`), there's a decent chance it builds with Cosmopolitan Libc right
now. Just try to build it! There are lots of possibilities.

[^windowspls]:

    I haven't tested running these executables on Windows, because I don't
    currently have a computer running Windows. Let me know what happens if you
    run the Actually Portable versions of `bash`, `curl`, or `git` on Windows.

[^cstandard]:

    Check out Section 6.8.4.2 of the C standard at [this pdf][cstandardpdf]. I
    believe a similar description of `switch` statements is provided for C11 and
    C17.

[^harmgoto]:
    
    Indeed, "`goto` is considered harmful" but I offer the following counters: I
    believe the simplest automatic way to rewrite `switch` statements (with
    `ifdef`s and fallthroughs mixed in) is to use `goto` statements, because
    otherwise you have to do some algebra in the condition of the `if`
    statement.  Also technically, I'm not telling _you_ to write `goto`
    statements, I'm telling `gcc` to convert your `switch` case labels into the
    `goto` target labels they actually are. So in a way I'm ensuring that you
    have to write less `goto` statements overall.

[^cppdiff]:

    Thus we have a difference between C/C++. As I found out recently while
    trying to patch `g++`, C++ does not seem to care that `struct`
    initializations have non-`static` constants: as far as I know, all of
    the initializations above are valid in C++ (as long as you have the order of
    members correct). Now `constexpr` though, that's a totally different story.

[^dblseg]:

    I'd like to introduce a new term into `gcc` plugin development, called the
    "segfault deadlock". A segfault deadlock is a situation where one alternates
    between segfaults from the compiler when compiling `foo.c` and segfaults
    from running the code generated by compiling `foo.c`.

[^readgcc]:

    Writing plugins is a good introduction into the `gcc` source; when debugging
    a plugin crash, I found it useful to read the implementations of functions
    that were declared in the `gcc` headers, eventually tracing the logic error
    back to my own code. The `gcc` source is C++, but I'd say it looks more like
    "C-with-templates", which are my favorite parts of C++ anyway. Also, `gcc`
    doesn't seem to use the C++ STL anywhere -- they have their own internal
    vector/hash table implementations. Lots to learn: I finally got a
    nonzero understanding of traits after using them for a hash table in `gcc`.


[^notcxx]:

    I specifically excluded C++ here because my arbitary expression substitution
    does not play well with the `constexpr` stuff in `g++11`. I don't get how
    `g++11` does `constexpr` -- I know where the code is, and I see some sort of
    recursive tree-walking happening, but I'm not as comfortable with it yet.
    Did you know `g++11` attempts to `constexpr` the `case` values within a
    `switch`, and that it's done differently compared to C? I didn't, I thought
    `switch` statements would be parsed in the same way for C and C++.


[tempcpy311]: https://github.com/ahgamut/cpython/tree/py311-custom
[cpy311]: https://github.com/ahgamut/cpython/tree/portcosmo
[plugin]: https://github.com/ahgamut/cosmo-gcc-plugin
[oldplugin]: https://github.com/ahgamut/cosmo-gcc-plugin/tree/3389ca1afd736af9ba187086d0896e0680312923
[pluginhere]: https://github.com/ahgamut/cosmo-gcc-plugin/commit/8ddce89551eadb6ea19ab843a71876cd75340edc
[pluginevent]: https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gccint/Plugin-API.html#Plugin-API
[patch]: https://github.com/ahgamut/gcc/tree/portcosmo-11.2
[buildpatch]: https://github.com/ahgamut/musl-cross-make
[superconf]: https://github.com/ahgamut/superconfigure
[lua]: https://github.com/jart/cosmopolitan/issues/61
[wren]: https://github.com/jart/cosmopolitan/issues/104
[janet]: https://github.com/jart/cosmopolitan/issues/114
[python]: https://github.com/jart/cosmopolitan/issues/141
[sqlite]: https://github.com/jart/cosmopolitan/pulls/162
[quickjs]: https://github.com/jart/cosmopolitan/issues/97#issuecomment-816831508
[cosmo]: https://github.com/jart/cosmopolitan
[jart]: https://github.com/jart
[pthreads]: https://github.com/jart/cosmopolitan/releases/tag/2.1
[cosmoverse]: https://justine.lol
[redbeanhn]: https://news.ycombinator.com/item?id=26271117
[faulthandler]: https://github.com/ahgamut/cpython/blob/master/Modules/faulthandler.c#L66
[ctor]: https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gcc/Common-Function-Attributes.html#Common-Function-Attributes
[cstandardpdf]: https://www.open-std.org/jtc1/sc22/wg14/www/C99RationaleV5.10.pdf
[gccint]: https://gcc.gnu.org/onlinedocs/gcc-11.2.0/gccint/
[ex1]: https://jongy.github.io/2020/04/25/gcc-assert-introspect.html
[ex2]: https://gabrieleserra.ml/blog/2020-08-27-an-introduction-to-gcc-and-gccs-plugins.html
[busyboxpls]:   https://github.com/mirror/busybox/blob/2d4a3d9e6c1493a9520b907e07a41aca90cdfd94/libbb/u_signal_names.c#L88
[ncurseswow]: https://github.com/mirror/ncurses/blob/master/ncurses/trace/lib_tracebits.c#L151
[thirdparty]: https://github.com/jart/cosmopolitan/tree/master/third_party
[rustpost]: {{< relref "2022-07-27-ape-rust.md"  >}}
