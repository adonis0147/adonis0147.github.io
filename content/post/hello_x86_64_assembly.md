---
title: "Hello x86-64 Assembly"
date: 2024-01-18T21:00:49+08:00
draft: false
summary: x86-64 Assembly on Linux
tags: ["x86-64", "Assembly", "Linux"]
categories: ["English"]
---

Recently, I practised x86-64 Assembly on Linux platform. This post records some examples and some issues I met during the process.

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

In the following program, the `_start` procedure calls `main` procedure.

```nasm
; Run:
;   nasm -f elf64 -g call.asm && ld call.o -o call
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
This violation may cause troubles. The following program is an example.

```nasm
; Run:
;   nasm -f elf64 -g call_seg_err.asm && ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 call_seg_err.o -o call_seg_err -lc
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
❯ ./call_seg_err
zsh: segmentation fault (core dumped)  ./call_seg_err
```

The rationale is that the `$rsp` is **NOT** 16-byte aligned just before calling the C function `puts`.  

We can resolve the issue like this.

```nasm
main:
    sub rsp, 8          ; to make the stack pointer 16-byte aligned

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    xor rax, rax        ; return 0

    add rsp, 8          ; restore the stack pointer

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

    xor rax, rax        ; return 0

    mov rsp, rbp        ; restore the stack pointer
    pop rbp             ; restore the frame pointer

    ret
```

We can also use the `LEAVE` instruction to restore the stack and frame pointers. Thus, we can simplify the `main` procedure like this.

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

The complete program is shown below.

```nasm
; Run:
;   nasm -f elf64 -g call_aligned.asm && ld -dynamic-linker /lib64/ld-linux-x86-64.so.2 call_aligned.o -o call_aligned -lc
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
    push rbp            ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp        ; save the stack pointer

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    xor rax, rax        ; return 0
    leave               ; restore the stack and frame pointers
    ret
```

# Use GCC to generate the executable

In the above examples, we use GNU linker `ld` to generate the final executable.
Alternatively, we can use GCC to help generate the executable which it is more convenient.

Consider the following example.

```nasm
; Run:
;   nasm -f elf64 -g call_gcc.asm && gcc -g call_gcc.o -o call_gcc
;
    global main
    extern puts

    section .rdata

message: db "Hello, world!", 0

    section .text

main:
    push rbp            ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp        ; save the stack pointer

    lea rdi, message    ; the first argument for C function puts
    call puts           ; call C function puts

    xor rax, rax        ; return 0
    leave               ; restore the stack and frame pointers
    ret
```

Unfortunately, I couldn't compile this code on my machine.

```shell
❯ nasm -f elf64 -g call_gcc.asm && gcc -g call_gcc.o -o call_gcc
/usr/bin/ld: call_gcc.o: relocation R_X86_64_32S against `.rdata' can not be used when making a PIE object; recompile with -fPIE
/usr/bin/ld: failed to set dynamic section sizes: bad value
collect2: error: ld returned 1 exit status
```

The GCC compiler was compiled with `--enable-default-pie` by default on Ubuntu 22.04.3 which causes this issue.
However, NASM doesn't provide a way to generate an object file compiled with `-fPIE`.

There are two workarounds.
1. Use `[rel symbol]` which is position-independent rather than just `symbol`.

    In the above example, the line `lea rdi, message` should be replaced by `lea rdi, [rel message]`.
2. Use `default rel` instead of modifying every addressing mode.

But, there is another issue occurring after applying the second workaround.

```shell
❯ nasm -f elf64 -g call_gcc.asm && gcc -g call_gcc.o -o call_gcc
/usr/bin/ld: call_gcc.o: warning: relocation against `puts@@GLIBC_2.2.5' in read-only section `.text'
/usr/bin/ld: call_gcc.o: relocation R_X86_64_PC32 against symbol `puts@@GLIBC_2.2.5' can not be used when making a PIE object; recompile with -fPIE
/usr/bin/ld: final link failed: bad value
collect2: error: ld returned 1 exit status
```

We can fix this issue by calling C function through the PLT (Procedure Linkage Table). Instead of calling `puts` directly, we call `puts wrt ..plt`.

The complete program is shown below.

```nasm
; Run:
;   nasm -f elf64 -g call_gcc.asm && gcc -g call_gcc.o -o call_gcc
;
    default rel             ; use relative addressing
    global main
    extern puts

    section .rdata

message: db "Hello, world!", 0

    section .text

main:
    push rbp                ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp            ; save the stack pointer

    lea rdi, message        ; the first argument for C function puts
    call puts wrt ..plt     ; call C function puts through PLT

    xor rax, rax            ; return 0
    leave                   ; restore the stack and frame pointers
    ret
```

# Pass parameters by stack

We use the stack to pass parameters of which the number is more than `6`. The following program is an example.

```nasm
; Run:
;   nasm -f elf64 -g call_printf.asm && gcc -g call_printf.o -o call_printf
;
    default rel             ; use relative addressing
    global main
    extern printf

    section .rdata

format: db "%d %d %d %d %d %d %d %d", 0xA, 0

    section .text

main:
    push rbp                ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp            ; save the stack pointer
    sub rsp, 8              ; expand the stack to a suitable place
                            ; in order to make it 16-byte aligned after pushing all the paramenters

    lea rdi, format         ; 1st argument
    mov rsi, 1              ; 2nd argument
    mov rdx, 2              ; 3rd argument
    mov rcx, 3              ; 4th argument
    mov r8, 4               ; 5th argument
    mov r9, 5               ; 6th argument
    push 8                  ; 9th argument
    push 7                  ; 8th argument
    push 6                  ; 7th argument
    xor rax, rax            ; because printf is varargs
    call printf wrt ..plt

    xor rax, rax            ; return 0
    leave                   ; restore the stack and frame pointers
    ret
```

The following program does the same thing.

```nasm
; Run:
;   nasm -f elf64 -g call_printf.asm && gcc -g call_printf.o -o call_printf
;
    default rel                 ; use relative addressing
    global main
    extern printf

    section .rdata

format: db "%d %d %d %d %d %d %d %d", 0xA, 0

    section .text

main:
    push rbp                    ; save the frame pointer and the stack pointer is 16-byte aligned now
    mov rbp, rsp                ; save the stack pointer
    sub rsp, 32                 ; expand the stack to a suitable place

    lea rdi, format             ; 1st argument
    mov rsi, 1                  ; 2nd argument
    mov rdx, 2                  ; 3rd argument
    mov rcx, 3                  ; 4th argument
    mov r8, 4                   ; 5th argument
    mov r9, 5                   ; 6th argument
    mov qword [rsp], 6          ; 7th argument
    mov qword [rsp + 8], 7      ; 8th argument
    mov qword [rsp + 16], 8     ; 9th argument
    xor rax, rax                ; because printf is varargs
    call printf wrt ..plt

    xor rax, rax                ; return 0
    leave                       ; restore the stack and frame pointers
    ret
```

* The stack grows **downwards**, so the place `$rsp + 8` is under the place `$rsp`.

* It is also worth noting that we should clear the `$rax` (perform `xor rax, rax`) before we call the C function `printf`.
This is because the C function `printf` uses a variable number of arguments and `$rax` specifies how many vector registers are used for the arguments.

# Some useful materials

* [NASM Tutorial](https://cs.lmu.edu/~ray/notes/nasmtutorial/)
* [System V ABI](https://wiki.osdev.org/System_V_ABI)
* [NASM Linux shared object error: Relocation R_X86_64_32S against '.data'](https://stackoverflow.com/questions/58106310/nasm-linux-shared-object-error-relocation-r-x86-64-32s-against-data)
* [Compile error: relocation R_X86_64_PC32 against undefined symbol](https://stackoverflow.com/questions/36007975/compile-error-relocation-r-x86-64-pc32-against-undefined-symbol/41322328)
* [Why is %eax zeroed before a call to printf?](https://stackoverflow.com/questions/6212665/why-is-eax-zeroed-before-a-call-to-printf)

