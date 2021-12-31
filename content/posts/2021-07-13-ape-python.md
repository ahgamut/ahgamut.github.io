---
categories: c
date: "2021-07-13T00:00:00Z"
slug: ape-python
title: Python is Actually Portable
aliases: "/c/2021/07/13/ape-python"
---

Back in February, I put together [Lua 5.4][lua_cosmo] using [Cosmopolitan
Libc][ghcosmo] as a quick proof-of-concept, and that led to Lua 5.4 being
[vendored][lua_vendor] as part of the Cosmopolitan repository on Github, along
with many other interesting developments.  It's pretty exciting to try and
compile well-known C projects using Cosmopolitan; the portability reward is
great motivation, and I get to observe the design, coding style, and build
system of high-quality codebases.

However, my initial plan with Cosmopolitan was not to compile Lua, it was to
compile Python. Since February, I've been trying to understand how Cosmopolitan
works, reading the repo code and submitting PRs occasionally, and finally I have
an actually portable version of Python 2.7.18[^py2eoltho] -- you can build it
from the repository [here][cosmo_py27].  It's not solid as Lua 5.4 because it
currently passes only a third of the regression tests, but most of the parts are
there.  For example, here's a GIF showing a simple [Flask webapp][flask] running
via the `python.com` APE.

![webapp](/images/ape-python.gif)

It's quite slow, but it works(tested on Debian Linux and Windows
10[^windows-pls]), and there's a lot of room for improvement. This post
describes the kinds of changes I made to get Python to work.

## Getting Python to compile ##

Python 2.7.18 is built via the [`configure` idiom][configure]: the tarball
provides a `configure` shell script which runs the necessary compatibility tests
for the host system/compiler. which then creates the Makefile for the actual
build, and fills `pyconfig.h` with the necessary `#define`s related to the
features available in the host system.

I added dummy headers like for Lua, and wrote a `superconfigure` script which
called `configure` with the flags for the Cosmopolitan
[amalgamation][cosmo_amalg], like below:

```sh
./configure --disable-shared \
    --without-threads --disable-ipv6 \
    CFLAGS="$COSMO_CFLAGS" \
    LDFLAGS="$COSMO_LDFLAGS" \
    LIBS="cosmopolitan.a"
make -j4
```

This started the build, but almost every source file had complaints about
functions being re-declared. For example, the Makefile assumed that `sinh` was
not available, and wrongly had `#define HAVE_SINH 0`, which caused it to use
Python's backup implementation. 

It turns out that the `CFLAGS` provided for the cosmopolitan amalgamation
clashed with [`AC_CHECK_FUNC`][checkfunc], which is what `configure` scripts use
to check if the host provides a particular function. At the time, the simplest
fix was to edit the `AC_CHECK_FUNC` part of  `configure` script to make it play
well, and it just worked[^alternAC]. 

After that, fixing the remaining errors was mostly straightforward: add a few
`#define`s or `#undef`s to avoid missing functions, ensure the Makefile
performed the linking correctly, find-and-replace for name
clashes[^undef-trick], and so on.


## Running the base interpreter ##

I tried running the `python.com` APE within the build directory, and it worked
without a hitch. But when I moved the executable somewhere, the interpreter
would start up and then exit with a failure `unable to import site`: Python
checks for `site.py` in `sys.path` at startup, which sets up the import context
for the rest of the standard library (stuff like which OS the APE is on for
`os.py`). Running `python.com -S` fixed the issue[^startup-flags], and later I
added a few relative folder paths to `sys.path` so that Python could search
there at startup.

One annoying issue I faced with the `python.com` REPL on Linux was I had to
press `Return` twice when I wanted to enter an empty line to the REPL. By
running `python.com -u` I found out this this was related to input buffering and
[`setvbuf`][setvbuf]: the interpreter was following Cosmopolitan's choice of
buffering IO, so I added a couple of lines to use unbuffered input by default.

When I tried `python.com -S` on Windows, the interpreter would start, but then
every line I entered at the REPL would cause a syntax error. I thought this was
also related to `setvbuf`, but this was because Windows [`cmd.exe` sends
CRLF][crlf] on pressing `Return`, and the interpreter expects only LF, so the CR
was read as incorrect syntax. It was straightforward to change the parser logic.

## Adding standard library modules ##

When the Makefile finishes building `python`, it immediately tries to call a
`setup.py` to build C extensions for the standard library. This was pretty
stupid: `setup.py` tries to build and link a bunch of shared objects to the
interpreter even though I've specified a static build (`--disable-shared`). I
found out it was doing this after I saw a successful build on the screen,
followed a deluge of (`unable to link`/`no dlopen`/`Rdynamic specified but also
static`) errors.

There is a nice way to compile C extensions into Python statically: via
`Modules/Setup`. This file follows a simple syntax to specify extensions and
their requirements, so that you can compile extensions before `setup.py` is
called.

```sh
*static*

# <name of extension> <source files in Modules folder> <includes> <links> -DSOME_FLAG
math mathmodule.c _math.c # -lm
array arraymodule.c

```

Calling `make` after changing `Modules/Setup` rebuilds the Makefile with the
necessary recipes for the specified extensions. You can read about
[`Modules/Setup` here][static-setup]. One nasty aspect of this is if the syntax
in `Modules/Setup` is wrong, the Makefile will be wrong, so you can't run `make`
even after you've fixed the error.

I was able to compile a lot of modules into APE using `Modules/Setup`: basic
modules like `cPickle`, `zlib` etc., modules that required external libraries
like `_sqlite`, `_bz2`, `readline`, `_ctypes`[^ctypes-fail] etc., and even
**external packages** that depend on simple C extensions, like
[`greenlet`][greenlet]. It all boils down to writing the correct recipe in
`Modules/Setup`, ensuring the necessary static libraries are available, and
checking that the glue code around the imports works correctly[^split-package].

The final annoyance with the stdlib python files was that I had to copy the
folder containing them alongside `python.com`, but this got solved because
Cosmopolitan allows the APE to be used as a valid ZIP file. I just zipped up
`Lib/` into the APE, *added `python.com` itself as an entry in `sys.path`*, and
Python's own [`zipimport` package][zipimport] handled the rest. 

## Using `pip` and external packages ##

After compiling most of the standard library packages, I still couldn't get
`ensurepip` to work, because it relied on threads, and the standard `threading`
module in Python would just raise an `ImportError` if it could not find the
`_thread` C extension. The fix was present within the stdlib: there is a module
called `_dummy_thread`, which spoofs `_thread` so that `threading` doesn't
complain[^thread-low]. After this, I was able to run `ensurepip` to install
`pip` and `setuptools` in `Lib/site-packages`.

However, running `python.com -m pip install` doesn't work because the Python APE
doe not currently have SSL support. I tried providing `http://pypi.org/simple`
as the index URL but `pip` was adamant in not letting me get my way.

I didn't want to figure out how to add SSL support to the APE (I expect it
should be possible[^ssl-possible]), so I just cheated and downloaded the
packages manually. The arrangement to add packages works like this:

```sh
mkdir -p Lib/site-packages

# specify python version to pip if necessary
/usr/bin/python2 -m pip download flask -t .

# wheels are just ZIPs, use unzip if pip complains
./python.com -m pip install flask*.whl -t ./Lib/site-packages

zip -qr ./python.com ./Lib/site-packages/
rm -rf ./*.whl ./Lib/site-packages/

./python.com -m flask --version
```

Of course, you would have recompile the APE if you wanted to add any C
extensions.

## Closing Notes ##

It is really convenient to have the same application work on
Linux/Windows/MacOS, but the techniques I have seen/used all have tradeoffs:
either for the developer in terms of design, testing, and safety/consistency, or
for the user in terms on application size and performance.

Cosmopolitan Libc does come with some tradeoffs as well (static compilation, C
codebases, no multithreading), but I think it is amazing that I can send a
`python.com` executable of a few megabytes as a zip file to someone, have them
run it without worrying about the OS, and provide them a simple webapp with a
backend that works even without an internet connection.

This successful `python.com` experiment unlocks many interesting directions: 

* Is it possible to tune the performance of the `python.com` APE? I don't expect
    it to get anywhere close to [`redbean`][redbean], but some speed
    improvements would be nice because Python web frameworks are established and
    easy to use.

* Is there room for a custom build system to produce a single-file size-optimized
 `python.com` APE webapp? We saw that it is possible to add C extensions, and
 even regular packages into the executable via `zip`. If there is a
 dependency resolver that accounts for stdlib imports, which then produces a
 `Modules/Setup` or some similar build script to compile extensions and add only
 the necessary libraries, that would be awesome[^pyobj].

* Regarding python packages with C extensions, the build process for adding them
    to the APE is rather unwieldy because of all the manual changes involved. Is
    there a way to automatically patch the imports, or better yet, compile
    Python C extensions as shared libraries with Cosmopolitan?

* I've managed to compile Lua, QuickJS, and now Python2.7 and 
[Python3.6][cosmo_py36]. Are there web-friendly languages that would benefit
more from a Cosmopolitan build? Maybe PHP, or Ruby, or even Go if the details
work out? Remains to be seen.


[^py2eoltho]:
    
    Did you click on this before reading the remainder of the post because you
    were wondering about Python 3?  I know Python 2.7 reached EOL last year, but
    the Python 2.7 codebase was easier to read, change, and debug, so I was able
    to get a better idea of where I had to make changes. I put together a Python
    3.6.14 APE (build it from the repo [here][cosmo_py36]), and it took much
    less time and experimentation because I knew exactly where to look.

[^windows-pls]:

    I originally uploaded this post like twelve hours, in the excitement of
    seeing a Flask webapp work on the APE. Unfortunately, I forgot to test
    on Windows before putting the post out, and Windows, true to its nature
    caused several annoyances which together took a couple of hours to sort
    out. However, I think it ought to work now.

[^alternAC]:

    I originally tried the complicated solution of writing a new configure
    script from scratch, but I got lost in the docs for `autoconf`, and resorted
    to the simpler fix of just editing the shell script. This issue only
    happened because I used dummy headers with the amalgamation; if you have the
    Cosmopolitan repo nearby, you can use `-isystem cosmopolitan/libc/isystem`
    instead of `-I dummy_headers -include cosmopolitan.h`, which would avoid
    confusing `configure`.

[^undef-trick]:

    I remembered this a few days back: if a name in your C file
    clashes with a header name somewhere, you can do something like:

    ```c

    #ifdef name
    #define name __name
    #undef name
    
    ```
    and avoid the name clash instead of doing a find-and-replace.

[^startup-flags]:

    Over the course of getting the APE to work, my muscle memory is to call
    `python -BESsuv`, which avoids a bunch of normal startup stuff and adds
    stderr information about imports. This is super useful when building python,
    but one time I was running a local script and couldn't figure out why it was
    failing before I saw that I had typed `python -BESsuv script.py` every
    single time.

[^ctypes-fail]:

    I was pleasantly surprised to find out that `_sqlite` and `_bz2` could be
    compiled, because I didn't expect SQLite or libbz2 would be easy to compile
    from source with Cosmopolitan. SQLite is now [vendored in the Cosmopolitan
    repo][sqlite-cosmo].

    On the other hand, I wanted `readline` because I like to have the
    (press-Up-arrow-for-previously-typed-line) in the REPL, but I didn't know
    how `terminfo` worked. `_ctypes` was also a disappointment because I had to
    compile `libffi` from source, and then I saw most of `_ctypes`' use comes
    from having `dlopen`.

[^split-package]:

    for example, `greenlet` contains a C extension called `_greenlet` which it
    refers to by doing things like the following in `__init__.py`:

    ```python
    
    from ._greenlet import foo
    from greenlet._greenlet import bar
    
    ```

    now `__init__.py` expects `_greenlet.so` to be present in the same
    directory, but that is not possible because the extension is compiled into
    the interpreter statically. So I had to manually change this to use `from
    _greenlet` without the dot. This also means I have to change the extension
    name in the `PyModuleDef` part of the C source code: it's not
    `greenlet._greenlet` anymore, it's just `_greenlet`.

[^thread-low]:

    This works because most libraries use only the high-level `threading` API to
    handle their threads. A notable counter-example is [Django][django]: I tried
    getting Django to work before figuring out how `_greenlet` could be
    compiled, but Django imports the low-level `_thread` API to handle things.

[^ssl-possible]:

    It is possible to have SSL support: you just have to build OpenSSL with
    Cosmopolitan Libc. I was able to build `libssl.a`, link it with `python.com`
    via `Modules/Setup`, and `pip` worked. However, I have not tested it with
    the OpenSSL test suite.

[^pyobj]:

    The Cosmopolitan repo builds `python.com` using [`PYOBJ.COM`][pyobj-com], a
    minimal Python executable that creates `.pyc` files and symbols for the
    linker to resolve dependencies.

[redbean]: https://redbean.dev/
[configure]: https://en.wikipedia.org/wiki/Configure_script
[checkfunc]: https://www.gnu.org/software/autoconf/manual/autoconf-2.69/html_node/Generic-Functions.html
[lua_vendor]: https://github.com/jart/cosmopolitan/issues/61#issuecomment-792359199
[cosmov1]: https://github.com/jart/cosmopolitan/releases/tag/1.0
[ghcosmo]: https://github.com/jart/cosmopolitan
[cosmo_py27]: https://github.com/ahgamut/cpython/tree/cosmo_py27
[cosmo_py36]: https://github.com/ahgamut/cpython/tree/cosmo_py36
[cosmo_amalg]: https://github.com/jart/cosmopolitan#getting-started
[crlf]: https://github.com/jart/cosmopolitan/issues/141#issuecomment-812918670
[setvbuf]: https://en.cppreference.com/w/c/io/setvbuf
[static-setup]: https://github.com/ahgamut/cpython/blob/cosmo_py27/Modules/Setup
[greenlet]: https://github.com/python-greenlet/greenlet/
[sqlite-cosmo]: https://github.com/jart/cosmopolitan/pull/162
[flask]: https://flask.palletsprojects.com/en/2.0.x/
[zipimport]: https://docs.python.org/2.7/library/zipimport.html
[django]: https://www.djangoproject.com/
[pyobj-com]: https://github.com/jart/cosmopolitan/blob/master/third_party/python/pyobj.c
[lua_cosmo]: {{< relref "2021-02-27-ape-cosmo.md"  >}}
