---
categories: c
date: "2022-10-23T00:00:00Z"
slug: debugging-c-with-cosmo
title: Debugging C With Cosmopolitan Libc
aliases: "/c/2022/10/23/debugging-c-with-cosmo"
---

[Cosmopolitan Libc][cosmo] provides a suite of debugging features that enhance
the C development experience: function call tracing, `gdb` integration, an
ASAN/UBSAN runtime, and more! A lot of fast and critical code is written in C --
If you're using software written in C, interfacing with C libraries, fixing bugs
in C code, or even rewriting C software in some other language, it helps to
understand what your C code is doing. Debugging isn't just a diaspora of
`printf` statements -- in this blog post, we'll look at how Cosmopolitan Libc
helps with debugging C, true and properly, using this [example repo][example-repo].

## An example program

Consider [`hex16.c`][example-part1]: the user specifies a file at the command
line, the program opens the file, prints (up to) the first 16 bytes as
hexadecimal values, and then prints the bytes read as an ASCII string. You can
find the entire code [here][example-repo].

```c
#include <stdio.h>
#define NUM_CHARS 16

void hex16(const char *filename) {
    FILE *fp = fopen(filename, "r");
    char res[NUM_CHARS];
    int i, c;

    printf("hex16.c -- reading file %s\n", filename);
    for (i = 0; i < NUM_CHARS; i++) {
        c = fgetc(fp);
        if (feof(fp) || c == -1) break;
        printf("0x%02x ", c);
        res[i] = (char)c;
    }
    res[i] = '\0';
    fclose(fp);
    printf("\n%s\n", res);
}

int main(int argc, char **argv) {
    hex16(argv[1]);
    return 0;
}
```

Let's build the above program with `make` and test it on a sample file:

```bash
make hex16.com
./hex16.com ./sample1.txt
# hex16.c -- reading file ./sample1.txt
# 0x63 0x6f 0x73 0x6d 0x6f 0x20 0x6c 0x69 0x62 0x63 0x0a 
# cosmo libc
```

But what happens if you don't provide a file or provide a nonexistent file?

```bash
./hex16.com
# Segmentation fault
./hex16.com ./missing.txt
# Segmentation fault
```

Hmm, `Segmentation fault` is not quite informative -- some part of the program
is causing a crash, but where?

## `gdb` and backtraces

With Cosmopolitan Libc, you can have [`gdb`][gdb] integration and detailed
backtraces for your C program in just one line: add `ShowCrashReports();` at the
start of the `main` function. Let's build [`hex16-backtrace.c`][example-part2]
which is just that:

```bash
make hex16-backtrace.com
# try with no arguments
./hex16-backtrace.com
# or try with a nonexistent file
./hex16-backtrace.com ./missing.txt
```

Now if you have [`gdb`][gdb] available on `$PATH`, you would get a TUI (terminal
user interace) showing the register contents and the backtrace of the crash,
like the image below:

![a gdb tui showing register info and the location of the crash](/images/debug-cosmo/gdb-cosmo.png)

Type `bt` and press `Enter` to view the backtrace in `gdb`, and exit by pressing
`Ctrl+D`. You can set up  a config for `gdb` from the Cosmopolitan Libc README
[here][cosmo-readme].

If `gdb` is not available, or if you're running the program as part of a test
script, you would get the backtrace in text, showing the register contents and
the backtrace of the crash:

```
error: Uncaught SIGSEGV (SEGV_MAPERR) on X550LD pid 256891 tid 256891
  ./hex16-backtrace.com
  EUNKNOWN/0/No error information
  Linux #1 SMP Debian 5.10.106-1 (2022-03-17) X550LD 5.10.0-13-amd64

RAX 0000000000000000 RBX 00007ffe5c5c9a70 RDI 000000000042dd48 ST(0) 0.0
RCX 0000000000000000 RDX 0000000000000000 RSI 0000000000000240 ST(1) 0.0
RBP 00007000007fff80 RSP 00007000007fff60 RIP 000000000040b698 ST(2) 0.0
 R8 0000000000000000  R9 0000000000000001 R10 0000000000000004 ST(3) 0.0
R11 0000000000000293 R12 0000000000000000 R13 000000000042dd46 ST(4) 0.0
R14 00007ffe5c5c9a80 R15 00007ffe5c5c9b90 VF PF ZF IF

XMM0  00000000000000000000000000000000 XMM8  00000000000000000000000000000000
XMM1  7865682f656c706d6178652d6f6d736f XMM9  00000000000000000000000000000000
XMM2  632d67756265642f6f6d736f632f6666 XMM10 00000000000000000000000000000000
XMM3  7574732f6d6168747561672f656d6f68 XMM11 00000000000000000000000000000000
XMM4  6f632e65636172746b6361622d363178 XMM12 00000000000000000000000000000000
XMM5  65682f656c706d6178652d6f6d736f63 XMM13 00000000000000000000000000000000
XMM6  2d67756265642f6f6d736f632f666675 XMM14 00000000000000000000000000000000
XMM7  74732f6d6168747561672f656d6f682f XMM15 00000000000000000000000000000000

0x000000000040b697: fixpathname at /home/jart/cosmo/libc/stdio/fopen.c:26
 (inlined by) fopen at /home/jart/cosmo/libc/stdio/fopen.c:62
0x00000000004098d0: hex16 at /home/gautham/stuff/cosmo/debug-cosmo-example/hex16-backtrace.c:5
0x000000000040291e: main at /home/gautham/stuff/cosmo/debug-cosmo-example/hex16-backtrace.c:22
0x00000000004029cb: cosmo at /home/jart/cosmo/libc/runtime/cosmo.S:77
0x0000000000402503: _start at /home/jart/cosmo/libc/crt/crt.S:103

10008004-10008004 rw-pa-   1x automap 64kB w/ 7872kB hole
10008080-100080ff rw-pa- 128x automap 8192kB w/ 96tB hole
6fe00004-6fe00004 rw-paF   1x g_fds 64kB
70000000-7000007f rw-Sa- 128x stack 8192kB
# 16mB total mapped memory
./hex16-backtrace.com 
```

From the backtrace, you now know that not providing a file causes a crash
involving `fopen`, and so update the program to check if the first parameter of
`fopen` (ie `argv[1]`) is NULL or not. Note that backtrace generation depends on
the `hex16-backtrace.com.dbg` file being located in the same directory as
`hex16-backtrace.com`, as it contains necessary debugging information.

Just knowing the backtrace of a crash helps reduce the time needed to fix an
error: when porting [`make` to Cosmopolitan Libc][make-pr], adding a
`ShowCrashReports` showed that `make` had some large `alloca` calls which were
causing a crash. The fix was to add a `STATIC_STACK_SIZE` call in the `main`
function, and then `make` could build the entire Cosmopolitan Libc monorepo.

## function call tracing with `--ftrace`

Sometimes a backtrace of the crash alone is not sufficient. Cosmopolitan Libc
allows you to log _every function call_ over the program's execution -- just
pass [`--ftrace`][ftrace] at the end of your program, like this:

```bash
./hex16.com ./missing.txt --ftrace
```
```
# truncated example
FUN 257012     1'567'864        48 &main
FUN 257012     1'573'031       112   &hex16
FUN 257012     1'578'356       160     &fopen
FUN 257012     1'612'346       336     &printf
FUN 257012     1'720'829       144     &fgetc
FUN 257012     1'726'479       176       &fgetc_unlocked
Segmentation Fault
```

* The first column indicates it is a function call
* the second column shows the process ID/thread ID calling the function
* the third column shows the approximate timestamp from the start of program
  execution
* the last column shows the name of the function that was called.

The `ftrace` output gets printed to `stderr` by default; it is useful to
redirect it to a file for later analysis. `ftrace` does not require
`ShowCrashReports()` at the start of the `main` function, though it still
requires the `hex16.com.dbg` for `hex16.com`. You can read more about how `ftrace`
works [here][ftrace].

`ftrace` also provides insight into program performance. I wrote a [Python
wrapper for `ftrace`][ftrace-pr] so I could examine how the CPython3.6 runtime
behaves. For example, here's how many calls it takes to add two values in
Python3.6 built with Cosmopolitan Libc:

```python
import cosmo
a = 1
b = 2
with cosmo.ftrace():
    c = a + b
```
```
&meth_dealloc
&PyFrame_BlockSetup 76
&object_dealloc 265
  &PyObject_Free 82
    &_PyObject_Free.isra.0 52
&PyDict_GetItem 333
  &lookdict_unicode_nodummy 157
&PyDict_GetItem 166
  &lookdict_unicode_nodummy 34
&PyNumber_Add 273
  &binary_op1 69
    &long_add 178
      &PyLong_FromLong 135
&PyDict_SetItem 273
  &insertdict 164
    &lookdict_unicode_nodummy 53
&PyFrame_BlockPop 320
&PyObject_CallFunctionObjArgs 249
  &object_vacall 40
    &_PyObject_FastCallDict 95
      &_PyCFunction_FastCallDict 53
        &_PyMethodDef_RawFastCallDict 86
          &_PyStack_AsTuple 61
            &PyTuple_New 121
          &FtracerObject_exit 199
```

The numbers on the right are an approximate measure of time each function takes,
which is pretty useful for a "slow" language like Python3.6[^pyfaster]. `ftrace`
helped write a [custom `sys.meta_path` importer][meta_path] for Python3.6 in
Cosmopolitan Libc -- I just moved code from Python into C until the size of the
ftrace logs stopped decreasing. 

## system call tracing with `--strace`

Sometimes tracing function calls is too much information, and you just want to
narrow down the error region from some logging information. If your program uses
a bunch of system calls, you can log them like functions, by passing
`--strace` at the end of your program invocation, like this:

```bash
./hex16.com ./missing.txt --strace
```
```
SYS 257304  43'627 bell system five system call support 171 magnums loaded on gnu/systemd
SYS 257304  99'854 mmap(0x700000000000, 8'388'608, PROT_READ|PROT_WRITE, MAP_STACK|MAP_ANONYMOUS, -1, 0) → 0x700000000000 (8'388'608 bytes total)
SYS 257304 892'097 getenv("TERM") → "xterm-256color"
SYS 257304 908'170 openat(AT_FDCWD, "./missing.txt", 0, 0) → -1 errno= 2
SYS 257304 932'876 write(1, u"hex16.c -- reading file ./missing.txt◙", 38) → 38 errno= 2
```

* The first column indicates it is a system call
* the second column shows the process ID/thread ID making the call
* the third column shows the approximate timestamp from the start of program
  execution
* the fourth column shows output of the system call along with errno values

Unlike `ShowCrashReports` and `ftrace`, the `--strace` flag does not require the
`hex16.com.dbg` file to be present.  Now you see that the `openat` resulted in
an `-1 errno 2` if the file doesn't exist, so let's update the code to check the
`FILE *` pointer before calling `fgetc`.  Thus you get the fixed program
[`hex16-fixed.c`][example-part3] below:

```c
#include <stdio.h>
#define NUM_CHARS 16

void hex16(const char *filename) {
    FILE *fp = fopen(filename, "r");
    char res[NUM_CHARS];
    int i, c;

    if (!fp) {
        printf("unable to read %s!\n", filename);
        return;
    }
    printf("hex16.c -- reading file %s\n", filename);
    for (i = 0; i < NUM_CHARS; i++) {
        c = fgetc(fp);
        if (feof(fp) || c == -1) break;
        printf("0x%02x ", c);
        res[i] = (char)c;
    }
    res[i] = '\0';
    fclose(fp);
    printf("\n%s\n", res);
}

int main(int argc, char **argv) {
    if (argc != 2) {
        printf("USAGE:\n\t%s FILENAME\n", argv[0]);
        return 1;
    }
    hex16(argv[1]);
    return 0;
}
```

## ASAN/UBSAN

The two errors in `hex16.com` have been fixed, so it's time to try another
testcase to see if there are any more. What happens if the input file contains
16 or more characters?

```bash
cat ./sample2.txt
# cosmopolitan libc
./hex16.com ./sample2.txt
# hex16.c -- reading file ./sample2.txt
# 0x63 0x6f 0x73 0x6d 0x6f 0x70 0x6f 0x6c 0x69 0x74 0x61 0x6e 0x20 0x6c 0x69 0x62 
# cosmopolitan lib2
./hex16.com ./sample2.txt
# hex16.c -- reading file ./sample2.txt
# 0x63 0x6f 0x73 0x6d 0x6f 0x70 0x6f 0x6c 0x69 0x74 0x61 0x6e 0x20 0x6c 0x69 0x62 
# cosmopolitan lib3
```

`hex16.com` has inconsistent behavior with `sample2.txt` -- sometimes it prints
extra garbage characters. But there is a `res[i] = '\0'` after the loop to
terminate the string, so what's going on? Let's build with the Cosmopolitan
Libc's ASAN/UBSAN runtime and find out:

```bash
make clean
make MODE=dbg hex16-backtrace.com
./hex16-backtrace.com ./sample2.txt
```
```
hex16-backtrace.c:16: ubsan error: 'int' index 16 into 'char [16]' out of bounds (tid 257624)
0x0000000000480c00: __ubsan_handle_out_of_bounds at /home/jart/cosmo/libc/intrin/ubsan.c:289
0x0000000000421658: hex16 at /home/gautham/stuff/cosmo/debug-cosmo-example/hex16-backtrace.c:16
0x0000000000402bf1: main at /home/gautham/stuff/cosmo/debug-cosmo-example/hex16-backtrace.c:22
0x0000000000402cfd: cosmo at /home/jart/cosmo/libc/runtime/cosmo.S:77
0x0000000000402543: _start at /home/jart/cosmo/libc/crt/crt.S:103
```

Ah -- when the file has more than 16 characters, the null terminator is written
_outside_ the buffer, causing a buffer overflow when the string is printed. Now
you can change the buffer to have one extra character, to handle this case,
leading to the now again-fixed [`hex16-ubsan.c`][example-part4].

Cosmopolitan's ASAN/UBSAN runtime adds a magical improvement to your C
development workflow[^ubsan1], for a tiny cost. Your build times increase only
marginally, you avoid any new arguments with the compiler, you still write your
tests as usual, and your binaries have almost the same performance. And of
course, your binaries will now have runtime memory safety. 

## In case of emergency (or lack of patience)

Sometimes you find an annoying bug that occurs only during a specific sequence
of events, and you just haven't figured it out. It's <u>not related to
ASAN/UBSAN</u> -- the runtime tells you some memory access is invalid, which is
good to know, but you don't know where it all started. The <u>system calls are
not shining light into the issue</u>, because it isn't related to how your
program interacts with the OS. The <u>function call sequences are too many</u>,
and mixed in due to multithreading, so it's difficult for you to make sense of
what's going on. You <u>start `gdb` and setup a dozen breakpoints</u> before
running the program, but it's too stop-start, and you don't have enough
experience or patience to poke at the right spots. In frustration, you reach for
the good old `printf` statements, but they don't work either! You have no tools,
because your tools haven't set up enough for your tools, like that bug where
Python hasn't set up `stdin` because the `encodings` module hasn't been
loaded[^python-bug] or the one where Rust's thread-local storage makes some
seemingly weird memory requests from the libc on startup[^rust-cosmo].

You're backed into a corner. How do you debug this seemingly impervious
[heisenbug][heisenbug]?

1. You imagine yourself as an `x86_64` chip (complete with power supply, RAM,
    and peripherals) and start [rubberducking][duck] each assembly instruction
    to hapless passerby until the end of time or until the bug is found,
    whichever is earlier.

2. You use [Cosmopolitan Libc's `kprintf`][kprintf].

[`kprintf`][kprintf] is the final tool in the Cosmopolitan Libc debugging
arsenal. Look at the [`kprintf` source code][kprintf-src] -- It's a lean
implementation of `printf` designed to work _everywhere_. It's what Cosmopolitan
Libc uses when the other functions want to write logs, even after the program
has crashed. So it has to work at all times. This is what happens if you try to
print some weird memory location using `kprintf`:

```c
kprintf("%s\n", 31415);
//!!7ab7
```

You can even clone the Cosmopolitan Libc monorepo, add `kprintf` calls in
Cosmopolitan Libc's internal functions, and test your code with your own
debug-customized libc! When all else fails, `kprintf` can get the job done,
provided you use it <strike>judiciously</strike> to print the value of every
variable after every statement in every function that your program has to call.

![bell curve meme with printf and kprintf](/images/debug-cosmo/kprintf.jpg)

## Closing Notes

Nobody writes perfect code in one try. Many eyes make bugs scarce. With
Cosmopolitan Libc, debugging becomes a game of Scotland Yard, instead of
blind-man's buff -- instead of stumbling around and stubbing your toes, you have
**five** different angles to understand what your program is doing and determine
where it starts going off the rails:

* you can have a simple backtrace with the register contents
* you can log all (or a subset of) the function calls to see which sequence possibly
  leads to a crash or a slowdown
* you can log all the system calls and examine the interaction between your
  program and the OS
* you can learn to use `gdb` and dissect your program with an ever-increasing number
  of breakpoints or similar wizardry
* when at your wit's end, you can revert to old habits, only this time you use
  `kprintf` instead of `printf` and gain superpowers

Underneath all the libraries, syntax, abstractions, and optimizations, we still
need to have some idea of what's going on. Writing programs is a lot more fun
when we can understand what the computer is actually doing. Detailed feedback
helps understand programs better, and quick feedback helps develop programs
faster.  Cosmopolitan Libc provides a set of debugging tools we can use to
enrich our understanding of the programs we write.

[^pyfaster]: 

    Python is undergoing a lot of performance improvements nowadays -- On
    2022-08-05, CPython released [`3.11.0rc1`][py311], and I ported that to
    Cosmopolitan Libc [here][cosmo_py311]. The 3.11 port doesn't have `ftrace`
    or the other goodies yet; it'll be interesting to see how `ftrace` gives
    insight into CPython's new "JIT-like" optimization tricks.

[^ubsan1]:
    
    A beautiful thing happens after a while developing with `MODE=dbg`: you're
    still as fast as before, but now you have a sixth sense, an intuition, or [a
    discipline of programming][adop] in C. Because the safety of UBSAN enforces
    only lenient bounds, you start writing UBSAN-tolerant code, but you also
    gain the confidence to gamble with performance tricks, knowing that UBSAN
    will save you if things go awry. I had never written a JSON parser before,
    but I wrote one in C, with [NaN-boxing and pointer manipulation][dabba], in
    a few afternoons. It passes around 90% of the [most comprehensive JSON Test
    suite][seriot], on `MODE=dbg` too.

[^python-bug]:
    
    This was a nasty bug that happened if you didn't provide a proper folder
    location for Python 3 to import its standard library. Here's my limited
    recall: Python 3 is Unicode-by-default. But it's not really Unicode when it
    starts up, it's still mostly ASCII-ish. Python does some locale checking
    before attempting to import `encodings.py`, and then sets up almost
    everything else. So if `encodings.py` is not found, Python gets stuck in
    this limbo and aborts. I could not connect the error messages to
    `encodings.py`, so I just `kprintf`'d every call in the import sequence
    until I found the fix.

[^rust-cosmo]:

    Don't `panic!` I said _seemingly_. There wasn't anything weird with how Rust
    initialized thread-local storage, because the Rust examples I wrote still
    worked on `glibc`. We eventually narrowed the bug down to a pointer
    initialization in `pthread_create` in Cosmopolitan Libc, and the fix was a
    [single line][stk-pr].  Cosmopolitan Libc does have `pthreads` support now,
    and it can run multithreaded [Python 3.11 scripts][py311-thread] and [Rust
    executables][rust-ape-example].

[example-repo]: https://github.com/ahgamut/debug-cosmo-example
[example-part1]: https://github.com/ahgamut/debug-cosmo-example/blob/main/hex16.c
[example-part2]: https://github.com/ahgamut/debug-cosmo-example/blob/main/hex16-backtrace.c
[example-part3]: https://github.com/ahgamut/debug-cosmo-example/blob/main/hex16-fixed.c
[example-part4]: https://github.com/ahgamut/debug-cosmo-example/blob/main/hex16-ubsan.c
[cosmo]: https://justine.lol/cosmopolitan
[cosmo-readme]: https://github.com/jart/cosmopolitan#gdb
[gdb]: https://www.sourceware.org/gdb/
[make-pr]: https://github.com/jart/cosmopolitan/pull/305
[ftrace-pr]: https://github.com/jart/cosmopolitan/pull/285
[meta_path]: https://github.com/jart/cosmopolitan/pull/425
[py311]: https://github.com/python/cpython/releases/tag/v3.11.0rc1
[cosmo_py311]: https://github.com/ahgamut/cpython/tree/cosmo_py311
[adop]: https://books.google.com/books/about/A_Discipline_of_Programming.html?id=MsUmAAAAMAAJ
[dabba]: https://github.com/ahgamut/cosmopolitan/tree/dabbajson/third_party/dabbajson
[seriot]: https://github.com/nst/JSONTestSuite
[heisenbug]: https://en.wikipedia.org/wiki/Heisenbug
[kprintf-src]: https://github.com/jart/cosmopolitan/blob/ef9776755ee3646029624fe30de5d58a3c03f6f6/libc/intrin/kprintf.greg.c
[kprintf]: https://justine.lol/cosmopolitan/documentation.html#kprintf
[duck]: https://en.wikipedia.org/wiki/Rubber_duck_debugging
[stk-pr]: https://github.com/jart/cosmopolitan/pull/606
[py311-thread]: https://github.com/ahgamut/cpython/blob/cosmo_py311/threading_example.py
[rust-ape-example]: https://github.com/ahgamut/rust-ape-example
[ftrace]: https://justine.lol/ftrace/
