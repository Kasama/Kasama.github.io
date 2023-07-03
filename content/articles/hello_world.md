---
categories: [Development, Linux]
date: 2023-06-30
description: How a hello world program gets executed on Linux
slug: hello_world
tags: [linux, c, syscall, shell, kernel, programming]
title: 'Hello World: A deeper dive'
---

As is tradition among programmers, the first thing you write is a kind of program called "Hello world". The goal is to simply print out a string to the console.

Take this simple C program:

```c
#include <stdio.h>

int main() {
    printf("Hello World!\n");

    return 0;
}
```

This is an example of a Hello World program in the C programming language, we can then compile it and run with

```bash
$ gcc -o main main.c
$ ./main
```

And the result should be the expected "Hello World!" output to the console.

```comment
Ah... it's so satisfying to have the program run and do exactly as expected, you feel powerful, in control of the machine, you are its guid...
```

Oh. But hold on a second... How _did_ that work? How did the program actually run? What happened when I typed `./main` there in the shell?

Well... It's complicated. And there are many lenses with which we can look at it. In this particular article we're going to take a look at a couple of them: One is the shell's perspective, what does the shell have to do to execute this process? The other is the Operating System's, how does it know where the code it needs to run is and how to get to it?

## The Shell

The shell, in this case `bash`, is a program where you can write commands and program names and it will execute them. We are interested in the way it runs commands by creating (spawning) a new process, then wait for it to exit.

So how does it do that?

The short answer is: It doesn't, but it asks the Kernel to with fork() -> execve() -> wait().

You can understand a lot about this process just by reading the syscall manual on Linux (`man 2 fork`, `man 2 execve`, `man 2 wait`), but let's go through it step by step.

### Asking the Kernel

The Kernel is the "center" of the Operating System, its code runs with the most privileges in the CPU ([Ring 0 on x86](https://en.wikipedia.org/wiki/Protection_ring)) and it can do _basically_ anything with the hardware.

Normal process normally run in what's called user-land, on the least privileged mode of the CPU, this means that to do anything that requires hardware interaction or some other operations, such as process control, it needs to _politely_ ask the Kernel by issuing a [syscall](https://en.wikipedia.org/wiki/System_call). The Kernel then checks for permissions among other things and either does what the process asked, or fails in a predictable manner.

### Forking

In Linux, the only way to create a new process is by being an existing process, then asking the Kernel to clone yourself. It seems a bit counter intuitive at first and it begs the question "Where did the first process come from?". We won't get to that now, but let's just say that the first one (init) is hand crafted as the last step in the booting process, then things sprawl out from there.

The `fork()` syscall is used by the shell process to ask for the Kernel to clone it. Then it runs some setup, such as arranging the input and outputs file descriptors among other things. At this point, it's ready to become the program we wish to run, our Hello World!

### Exec-ing

The process we have after forking is an exact copy (well... [with some differences](https://man7.org/linux/man-pages/man2/fork.2.html)) of the one that called `fork()`. So how can it run our Hello World program?

A process in Linux can also ask for the Kernel to completely replace it with something else, something new, and shinny, and better, and Hello Worldy. The `execve()` syscall does exactly that. It takes an argument with the program to run and arguments, then it replaces the current process with that program (if it's a script it's a bit different, [read the manual](https://man7.org/linux/man-pages/man2/execve.2.html) for more info).

Now the child program can go about it's business and do whatever it wants to (Print "hello world" to the console) and exit at some point in the future.

### Waiting

We ran the process in the shell with the expectation that it would run to completion and we would get the shell back afterwards. For that to be possible, the shell needs a way to be notified when the child process is done. As we've seen before, that user-land process doesn't have access to process control, so it asks the Kernel to make it `wait()` until the child process exits (it can do more than that, again [read the manual](https://man7.org/linux/man-pages/man2/wait.2.html)).

After the child's state is changed to terminated, that means that the parent shell can show itself to us again and receive new commands. Then it's all done and we have "Hello World!" printed on the screen.

In the end, the flow looks a bit like this:

```goat
          +---------------+
          |               |
          | Shell Process |
          |               |
          +---------------+
               fork()
                 /\
                /  \
               /    \
+-------------v-+  +-v-------------+
|               |  |               |
| Shell Process |  | Shell Process |
|   (Parent)    |  |    (Child)    |
+---------------+  +---------------+
        |                  |
        |               execve()
        |                  |
        |           +------v--------+
        | wait()    |               |
        +--------->||  Hello World  |
                    |               |
                    +---------------+
```

## The child process (Hello world)

In the exec section, we saw that the child process becomes our hello world program, which then goes about its business, but how does it do that?

Let's take a look at the start of `main` file generated by the `gcc` compiler we called earlier

```txt
$ xxd main
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0300 3e00 0100 0000 4010 0000 0000 0000  ..>.....@.......
00000020: 4000 0000 0000 0000 c034 0000 0000 0000  @........4......
00000030: 0000 0000 4000 3800 0d00 4000 1e00 1d00  ....@.8...@.....
...
```

Wow, those certainly are numbers... Let's look at what they mean. In the [ELF format spec](https://refspecs.linuxbase.org/elf/elf.pdf) we can see that the file starts with a header with some information. In figure 1-3 we are shown this struct.

```c
#define EI_NIDENT 16
typedef struct {
    unsigned char e_ident[EI_NIDENT];
    Elf32_Half e_type;
    Elf32_Half e_machine;
    Elf32_Word e_version;
    Elf32_Addr e_entry;
    Elf32_Off e_phoff;
    Elf32_Off e_shoff;
    Elf32_Word e_flags;
    Elf32_Half e_ehsize;
    Elf32_Half e_phentsize;
    Elf32_Half e_phnum;
    Elf32_Half e_shentsize;
    Elf32_Half e_shnum;
    Elf32_Half e_shstrndx;
} Elf32_Ehdr;
```

If we map the bytes in the hex dump to this struct, we get

```txt
;; e_ident
00000000                 7f45 4c46               ; File format: \x7FELF
00000004                 02                      ; File class: 64-bit
00000005                 01                      ; Data encoding: little-endian
00000006                 0100 0000               ; File version: UNIX System V ABI
0000000A                 0000 0000 0000          ; Padding
;; e_type
00000010                 0300                    ; File type: Shared object
;; e_machine
00000012                 3e00                    ; Machine: x86-64
;; e_version
00000014                 0100 0000               ; File version
;; e_entry
00000018                 4010 0000               ; Code entry point
;; e_phoff
00000020                 4000 0000 0000 0000     ; Program Header (PH) offset in file
;; e_shoff
00000028                 C034 0000 0000 0000     ; Section Header (SH) offset in file
;; e_flags
00000030                 0000 0000               ; Processor-specific flags
;; e_ehsize
00000034                 4000                    ; ELF header size
;; e_phentsize
00000036                 3800                    ; PH entry size
;; e_phnum
00000038                 0D00                    ; Number of entries in PH
;; e_shentsize
0000003A                 4000                    ; SH entry size
;; e_shnum
0000003C                 1E00                    ; Number of entries in SH
;; e_shstrndx
0000003E                 1D00                    ; SH entry index for string table
```

Okay, quite a bit to unpack here. When going to run the ELF executable, the OS will parse this header and map the sections to memory. Then it will jump execution to `e_entry` and hand control over to the code there. So `e_entry` should be the address of our `main` function, right?

```note
To make my life easier, I'll not translate everything by hand, but will use [IDA pro](https://hex-rays.com/ida-pro/) to do that for me
```

Let's look into it, then. Going to `4010 0000`, which is address `0x0000000000001040` (64-bit [little-endian](https://en.wikipedia.org/wiki/Endianness), remember?) we'll see this
```asm
; ===========================================================================

; Segment type: Pure code
; Segment permissions: Read/Execute
_text           segment para public 'CODE' use64
                assume cs:_text
                ;org 1040h
                assume es:nothing, ss:nothing, ds:_data, fs:nothing, gs:nothing

; =============== S U B R O U T I N E =======================================

; Attributes: noreturn fuzzy-sp

                public _start
_start          proc near               ; DATA XREF: LOAD:0000000000000018↑o
; __unwind {
                endbr64
                xor     ebp, ebp
                mov     r9, rdx         ; rtld_fini
                pop     rsi             ; argc
                mov     rdx, rsp        ; ubp_av
                and     rsp, 0FFFFFFFFFFFFFFF0h
                push    rax
                push    rsp             ; stack_end
                xor     r8d, r8d        ; fini
                xor     ecx, ecx        ; init
                lea     rdi, main       ; main
                call    cs:__libc_start_main_ptr
                hlt
; } // starts at 1040
```

Hmm... 🤔

That doesn't look like our `main` function. It says its name is `_start`, but we didn't write no `_start` function. In fact, it seems to run some setup code, then call our `main` function.

What's happening here is that libc is including some handy setup for us, some stack protections, setting up the [`at_exit`](https://man7.org/linux/man-pages/man3/atexit.3.html) handler and some other stuff. Eventually it calls our `main` function that does the actually important work (printing Hello World).

```comment
Can we get rid of all that nonsense? I just want to print Hello World!

```

We sure can! We can use the gcc flag `-nostartfiles` to make sure it doesn't generate a `_start` function for us, which of course, means we need to provide one.

```c
#include <stdio.h>

void _start() {
    printf("Hello World!\n");
}
```

Then we can compile it and run.

```sh
$ gcc -o main -nostartfiles main.c
$ ./main
hello world
[2]    2107712 segmentation fault (core dumped)  ./main
```

Uh oh... We got a segmentation fault there, why is that!? Let's look at the generated executable once again.

```asm
; ===========================================================================

; Segment type: Pure code
; Segment permissions: Read/Execute
_text           segment byte public 'CODE' use64
                assume cs:_text
                ;org 1020h
                assume es:nothing, ss:nothing, ds:LOAD, fs:nothing, gs:nothing

; =============== S U B R O U T I N E =======================================

; Attributes: bp-based frame

                public _start
_start          proc near               ; DATA XREF: LOAD:0000000000000018↑o
; __unwind {
                push    rbp
                mov     rbp, rsp
                lea     rax, s          ; "hello world"
                mov     rdi, rax        ; s
                call    _puts
                nop
                pop     rbp
                retn
; } // starts at 1020
```

Ok, so we no longer have a `main` function as expected, but we are trying to `retn` in the end of the `_start()` function there, but we haven't setup a place to go to, so it's tyring to jump to `0x0000000000000001` (the value `rsp` is currently pointing to). This is clearly a violation, which then incurs a `SIGSEGV` to the application.

It is expected that the application doesn't return, but rather calls a syscall to notify the OS that it has finished. The `exit()` syscall is the one that does that (`execve()` also works, but then it would be the responsibility of the replacement process to exit properly).

[`exit()`](https://man7.org/linux/man-pages/man2/exit.2.html) does some more things as well, such as cleaning up open file descriptors, re-parenting child processes, etc. It makes sure that there's no trash lying around after the program is done, so let's add it to our code.

```c
#include <stdio.h>
#include <stdlib.h>

void _start() {
    printf("Hello World!\n");
    exit(0);
}
```

```sh
$ gcc -o main -nostartfiles main.c
$ ./main
hello world
```

Success! It prints hello world and doesn't crash anymore.

Now, after it exits the OS will clean up everything and keep a small reference to the dead process (called a Zombie). This Zombie must be reaped by the parent process with the `wait()` syscall, which is why the shell does that, so it can reap the child and get its status code.

## Conclusion

Now you should have a good idea of how a process actually runs in Linux.

In this example we used C to generate the executable, but the same happens for any language that compiles to an ELF executable. For interpreted script languages, the interpreter is likely an ELF file, then the `execve()` syscall will try to load the header, and will find a shebang (`#!`) in the beginning, which will cause it to load the interpreter instead.

As a follow-up, we could also try to remove libc from our program completely, by calling the `exit()` and `write()` syscalls manually instead of using stdlib.h and stdio.h, but that requires some `asm` blocks, which would be platform specific and would require us to be careful with the registers we cobble and what not, but it's a fun experiment.
