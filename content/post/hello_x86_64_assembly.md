---
title: "Hello x86-64 Assembly"
date: 2024-01-18T21:00:49+08:00
draft: true
summary: x86-64 Assembly on Linux
tags: ["Assembly", "Linux"]
categories: ["English"]
---

Recently, I practised x86-64 Assembly on Linux platform. This post records some example programs and some issues I met in the process.

# Environment

1. Ubuntu 22.04.3 LTS 
2. NASM version 2.15.05
3. GNU ld 2.38
4. gcc 11.4.0

# Invoke syscall

There are some ways to invoke syscall and I just want to mention the following two ways here.

## 1. INT 0x80 (32-bit, _deprecated_)

```nasm
; Run:
;   nasm -f elf64 -g int80h.asm && ld int80h.o -o int80h
;
    global _start

    section .rdata

message: db "Hello, world!", 0xA
length: equ $ - message

    section .text

_start:
    mov eax, 4          ; call write
    mov ebx, 1          ; file descriptor 1 (stdout)
    lea ecx, message    ; the string to write
    mov edx, length     ; the length of the string
    int 0x80            ; invoke the syscall

    mov eax, 1          ; call exit
    xor ebx, ebx        ; error code 0
    int 0x80            ; invoke the syscall
```

* The number of the syscall can be referred to `/usr/include/x86_64-linux-gnu/asm/unistd_32.h`.
* The calling convention can be referred to [linux/arch/x86/entry/entry_32.S](https://github.com/torvalds/linux/blob/296455ade1fdcf5f8f8c033201633b60946c589a/arch/x86/entry/entry_32.S#L780).

## 2. SYSCALL (default in 64-bit, _recommended_)

```nasm
; Run:
;   nasm -f elf64 -g syscall.asm && ld syscall.o -o syscall
;
    global _start

    section .rdata

message: db "Hello, world!", 0xA
length: equ $ - message

    section .text

_start:
    mov rax, 1          ; call write
    mov rdi, 1          ; file descriptor 1 (stdout)
    lea rsi, message    ; the string to write
    mov rdx, length     ; the length of the string
    syscall             ; invoke the syscall

    mov eax, 60         ; call exit
    xor rdi, rdi        ; error code 0
    syscall             ; invoke the syscall
```

* The number of the syscall can be referred to `/usr/include/x86_64-linux-gnu/asm/unistd_64.h`.
* The calling convention can be referred to [linux/arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/296455ade1fdcf5f8f8c033201633b60946c589a/arch/x86/entry/entry_64.S#L68).

# System V ABI (x86-64 calling convention)

We should be familiar to the calling convention when we coding in Assembly. [This link](https://wiki.osdev.org/System_V_ABI) is very helpful.

> This is a 64-bit platform. The stack grows downwards. Parameters to functions are passed in the registers rdi, rsi, rdx, rcx, r8, r9, and further values are passed on the stack in reverse order.
> Parameters passed on the stack may be modified by the called function. Functions are called using the call instruction that pushes the address of the next instruction to the stack and jumps to the operand.
> Functions return to the caller using the ret instruction that pops a value from the stack and jump to it. The stack is 16-byte aligned just before the call instruction is called.

> Functions preserve the registers rbx, rsp, rbp, r12, r13, r14, and r15; while rax, rdi, rsi, rdx, rcx, r8, r9, r10, r11 are scratch registers.
> The return value is stored in the rax register, or if it is a 128-bit value, then the higher 64-bits go in rdx.

In the following snippet, the `_start` procedure calls `main` procedure.

```nasm
; Run:
;   nasm -f elf64 -g calling.asm && ld calling.o -o calling
;
    global _start

    section .text

_start:
    call main

    mov rdi, rax        ; error code
    mov eax, 60         ; call exit
    syscall             ; invoke the syscall

main:
    ret
```

The `CALL` instruction pushes the `$rip` to the stack and `RET` instruction pops a value from the stack and jump to it. We can verify this by GDB.

```shell
Program stopped.
0x0000000000401000 in _start ()
(gdb) disassemble
Dump of assembler code for function _start:
=> 0x0000000000401000 <+0>:     call   0x40100f <main>
   0x0000000000401005 <+5>:     mov    %rax,%rdi
   0x0000000000401008 <+8>:     mov    $0x3c,%eax
   0x000000000040100d <+13>:    syscall
End of assembler dump.
(gdb) x/a $rsp
0x7fffffffe1d0: 0x1
(gdb) stepi
0x000000000040100f in main ()
(gdb) disassemble
Dump of assembler code for function main:
=> 0x000000000040100f <+0>:     ret
End of assembler dump.
(gdb) x/2a $rsp
0x7fffffffe1c8: 0x401005 <_start+5>     0x1
(gdb) info frame
Stack level 0, frame at 0x7fffffffe1c8:
 rip = 0x40100f in main; saved rip = 0x401005
 Arglist at 0x7fffffffe1c0, args:
 Locals at 0x7fffffffe1c0, Previous frame's sp is 0x7fffffffe1d0
 Saved registers:
  rip at 0x7fffffffe1c8
(gdb) nexti
0x0000000000401005 in _start ()
(gdb) x/a $rsp
0x7fffffffe1d0: 0x1
(gdb) x/2a $rsp - 8
0x7fffffffe1c8: 0x401005 <_start+5>     0x1
```

It is worth noting that the stack is **16-byte aligned** just before the call instruction is called.  

In the above example, the `$rsp` is **16-byte aligned** (`0x7fffffffe1d0`) before calling the `main` procedure.
However, after performing the `CALL` instruction, the `$rsp` is **8-byte aligned** (`0x7fffffffe1c8`) due to the `$rip` was pushed to the stack.
This violation may cause troubles. The following snippet is an example.

```nasm
; Run:
;   nasm -f elf64 -g calling_seg_err.asm && ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 calling_seg_err.o -o calling_seg_err -lc
;
    global _start
    extern puts

    section .rdata

message: db "Hello, world!", 0

    section .text

_start:
    call main

    mov rdi, rax        ; error code
    mov eax, 60         ; call exit
    syscall             ; invoke the syscall

main:
    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    xor rax, rax        ; return 0
    ret
```

In my machine, I got a segmentation fault when I ran the program.

```shell
❯ ./calling_seg_err
zsh: segmentation fault (core dumped)  ./calling_seg_err
```

The rationale is that the `$rsp` is **NOT** 16-byte aligned just before calling the C function `puts`.  

We can resolve the issue like this.

```nasm
main:
    sub rsp, 8          ; to make the stack pointer 16-byte aligned

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    add rsp, 8          ; restore the stack pointer

    xor rax, rax        ; return 0
    ret
```

> Optionally, functions push rbp such that the caller-return-rip is 8 bytes above it, and set rbp to the address of the saved rbp.
> This allows iterating through the existing stack frames. This can be eliminated by specifying the -fomit-frame-pointer GCC option.

Alternatively, we can just push `$rbp` to the stack to make the `$rsp` 16-byte aligned.
What's more, we usually assign `$rsp` to `$rbp` which allows iterating through the existing stack frames.

```nasm
main:
    push rbp            ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp        ; save the stack pointer

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    mov rsp, rbp        ; restore the stack pointer
    pop rbp             ; restore the frame pointer

    xor rax, rax        ; return 0
    ret
```

We can also use the `LEAVE` instruction to restore the stack and frame pointers. Thus, we can simply the `main` procedure like this.

```nasm
main:
    push rbp            ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp        ; save the stack pointer

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    xor rax, rax        ; return 0
    leave               ; restore the stack and frame pointers
    ret
```
