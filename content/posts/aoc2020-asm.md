+++
title = "Completing Advent of Code 2020 Day 1 in x86-64 Assembly"
date = 2021-01-14

[taxonomies]
categories = ["Programming"]
tags = ["assembly", "linux", "aoc"]
+++

I think the Advent of Code challenges are a wonderful opportunity to experiment with new programming languages.
As I have been trying to learn more about operating systems and the low level operations of computers, I thought
it would be an interesting exercise to try to complete some of the Advent of Code 2020 challenges in x86-64bit
assembly on Linux.

<!-- more -->

# Preface

I will be going through my assembly solution to the Advent of Code 2020 Day 01 challenge.
Along the way, I will attempt to explain what assembly is, why it is important, and how the code works.

I expect that you will already have some experience with programming. This doesn't have to be low level
programming, or assembly, or anything like that, but that would help tremendously. If you have a decent
amount of experience in programming languages such as Python or Javascript, you may have to go slowly
and Google some things, but you should be able to follow along.

If you would like to follow along with me, you will need
* the [nasm](https://www.nasm.us/) x86 assembler
* the [gcc](https://gcc.gnu.org/) compiler
* the [make](https://www.gnu.org/software/make/manual/make.html) build tool

I leave the installation of these tools to you.

# Introduction

To execute a program, your CPU accepts machine code instructions. These are raw bytes that instruct the
CPU to perform one of the low level operations it knows how to do -- such as access memory, perform
basic arithmetic, move data into registers, etc.

While programmers did write raw machine code in some of the earliest days in computing, almost no one
writes raw machine code today. Machine code is incredibly difficult for humans to read, write, and understand
because it is merely a sequence of numbers encoded in binary that is sent to the CPU. Imagine encoding the
English alphabet with the numbers 1-26, and trying to write a paper using those numbers in place of letters.
While it is possible, it would take an exorbitant amount of time and likely be fraught with errors.

These early computer scientists soon developed **assembly** as an abstraction on top of machine code. Instead
of writing the sequence of numbers that would encode machine code instructions, code would be written with
text that humans could better read and understand. Each line would encode one instruction and any arguments
that instruction would accept. Labels were also added so that code could refer to certain parts of the program
in a simple way, allowing programmers to more easily implement loops, functions, conditional statements, etc.

However, assembly cannot be executed by the CPU -- the CPU can only execute machine code. Thus, assembly must
be *assembled* into machine code by an assembler. An assembler is essentially a compiler from assembly to machine code.
This machine code can then be sent to the CPU and executed.

As assembly is only a light abstraction over machine code, it is highly processor-specific and low level. That means
that assembly code must take certain features of the CPU into account to work properly (or at all), such as
the architecture, registers, addressing modes, data types, and more.

We will be writing code for the x86 architecture, targeting CPUs with 64bit registers. This is usually succinctly
described as x86-64 assembly code.

Furthermore, as we are not writing assembly for a bare metal application, we rely on interfacing with the operating
system to perform many operations, such as opening, reading from, writing to, and closing files. The interface to
perform these operations through the operating system are operating system specific, so we will be targeting Linux
in this article.

# Assembly Basics

We will first review some preliminary aspects of assembly code that are required to read or write any assembly.

## Registers

Much of assembly code performs operations with registers. Registers are small memory locations within the CPU that
store the data that the CPU performs operations on. For instance, when performing an add operation, one of the
operands must be CPU register that the CPU will save the final sum to.

CPU registers are typically either 8bit, 16bit, 32bit, or 64bit. Most modern desktop processors today are 64bit,
and as stated before, we are writing 64bit assembly code.

These registers are all referred to with special -- and somewhat confusing -- names in assembly code.

The table [here](https://www.cs.uaf.edu/2017/fall/cs301/lecture/09_11_registers.html) lists all of the
x86-64bit registers.

Note that although these registers are 64bit, you can specifically access smaller portions of the registers.
For instance, the lower 32bit half of the `rax` register is `eax`; the lowest 16bit fourth is `ax`;
the top 8bit half of `ax` is `ah`, and the low 8bit half of `ax` is `al`. The table linked above also lists
the names of all of these subparts of each 64bit register.

Note that you cannot access the high 32bits of `rax` directly, nor the high 16bits of `eax`.
You can do this with bit operations, but you cannot access them directly.

We can move data into registers with the `mov` class of instructions.

```nasm
mov rax, 0xdeadbeef ; move the number 0xdeadbeef into rax
mov eax, 42         ; move the number 42 into eax, the lower 32bits of rax
mov ax, 0xff        ; move the number 0xff into ax, the lowest 16bits of rax
mov ah, 0xf         ; move the number 0xf into ah, the high 8bits of ax
mov al, 0xa         ; move the number 0xa into al, the low 8bits of ax
```

## Instructions

We should now discuss instructions.

Every line of an assembly program encodes exactly one assembly instruction. An instruction is an
one operation that a CPU executes. Most instructions additionally take one or more *operands*,
which are arguments that specify the operation of the instruction. For instance, the `mov`
instruction showed above takes a register as its first operand and a number as its second operand.
This encodes the operation of moving the number into the register. There are different forms of
many instructions, however, that accept different operands; for instance, each line above is a
different form of `mov`, as it accepts a different size of register. This means that the machine
code number of the `mov` instruction is different on each line. Thus, each form of an instruction
is encoded by a different number in machine code.

We do not have to worry too much about the different forms of instructions and their machine
code numbers normally, thankfully. You just have to know that there are many instructions and each
instruction can take none, one, or more operands.

The `mov` instruction discussed above, generally, moves data from a source to a destination of equal
size in the form `mov [dest] [src]`.

### Syntax

As a quickaside, I will mention that the assembly *syntax* we are using is only one of many. In
other syntaxes, such at AT&T syntax, the `mov` instruction would be of form `mov [src] [dest]`.

We are using the NASM assembly syntax because it is simple and easy to read. Intel syntax is also
very common and looks fairly similar to NASM syntax. AT&T syntax is very common as well, but
it looks very different from NASM syntax and personally, I find it to be very noisy and difficult
to read.

Most written assembly nowadays seems to use NASM syntax and the NASM assembler, as we are using.

### More Instructions

Now that we know what an instruction is, let us look at some useful assembly instructions
that we will make use of:
* `mov [dest] [src]` : move data from the source to the destination
* `lea [dest] [mem]` : loads a memory address into the destination register
* `xor [src] [dest]` : xor the data in the `src` register with the value given as `dest`
* `add [dest] [src]` : add the number, register, or number in memory to the destination register
* `sub [dest] [src]` : subtract the number, register, or number in memory from the destination register
* `imul [src]` : do a signed multiplication between `rax` and the `src` register.
    * Note that there are other forms of `imul` for smaller register sizes; if you use a 32bit,
        16bit, or 8bit register as the `src`, then the corresponding size of the `rax` register
        will be used in the signed multiplication
* `mul [src]` : do an unsigned multiplication between `rax` and the `src` register
    * The size correspondence as explained right above also applies here
* `cmp [src] [dest]` : compare the value in the `src` register to the value given in `dest` and
    set the corresponding flags
    * We will discuss flags in the next subsection.
* `jmp [label]` : unconditionally jump to the given label
* `jl [label]` : jump to a given label if the `B` (below) flag is set
* `jg [label]` : jump to a given label if the `A` (above) flag is set
* `je [label]` : jump to a given label if the `E` (equal) flag is set
* `jne [label]` : jump to a given label if the `NE` (not equal) flag is set
* `call [label]` : call a function called `label`
* `syscall` : Perform a `syscall`.
    * We will discuss what a syscall is later.

### Flags

When you perform some operations, such as a `cmp` operation, the CPU will set some flags for you.
These flags cannot be read or written to directly, but can be used for control flow with some
instructions. For instance, most languages have an `if` statement to select between two
code branches to execute depending on some condition. The following Python code,

```python
a = 2
if a > 1:
    a += 1
else:
    a -=1
```

would translate to the following assembly,

```nasm
    mov rax, 2      ; move the number 2 into rax
    cmp rax, 1      ; compare the values of rax and 1
    jle sub_one     ; if rax is less than or equal to 1, jump to sub_one
    add rax, 1      ; add one to rax -- the 'if' path
    jmp exit        ; jump to exit
sub_one:            ; the sub_one label
    sub rax, 1      ; subtract one from rax -- the 'else' path
exit:               ; the exit label
```

If `rax` is less than or equal to `1`, then the `jle [label]` will execute and jump to the
`sub_one` label. Otherwise, this instruction will be passed over, `1` will be added to `rax`,
and the program will jump to the `exit` label to skip the subtraction instruction.

You don't have to worry about what flags there are and how they are set; just treat the
`cmp` instruction as a comparison between two numbers, and follow that with whatever
jump-family instruction corresponds to the comparison you want to make. That is,
* `jl` corresponds to `<`
* `jle` corresponds to `>=`
* `jg` corresponds to `>`
* `jge` corresponds to `>=`
* `je` corresponds to `==`
* `jne` corresponds to `!=`
Thus, using the `cmp` and `jmp` instructions, you can implement conditional control flow to jump to
different code branches depending on the value of some register.

### Syscalls

I mentioned before that to perform many common operations, such as opening, reading from,
writing to, and closing files, we will have to interface with the operating system. This is
because the operation manages all of the hardware in a computer, including storage, and thus
the operating system manages the filesystem. Thus, if we want to interact with files, we must
work with the operating system. Whenever you open a file in a higher level language, it is
ultimately interfacing with the operating system in this way.

As we are working with Linux in our code, we will use the *syscall* interface. This is
ultimately rather simmple. We will just have to move some data into specific registers
to tell the *Linux kernel* what operation we want to perform, as well as supply data
to perform that operation, and then we use the `syscall` instruction.

Each syscall has a corresponding number. The ones that we will use are:
* `read` -- 0 : read from a file
* `write` -- 1 : write to a file
* `open` -- 2 : open a file
* `close` -- 3 : close a file

We move the number of the syscall into `rax`, move other arguments into other specified
registers, and then use `syscall` to perform the operation.

Think of a syscall as a function -- just a special function that the Linux kernel executes.

## Memory

With the basics of assembly instructions out of the way, we will discuss the basics of
memory that we will need.

We will have to allocate memory in our program for arrays, strings, numbers, and more.
However, we won't deal with dynamically allocating memory (such as with `malloc`)
or anything like that. We will just bake the memory we need into the executable program
itself through the equivalents of global variables.

We can allocate strings and numbers in NASM using the following syntax,

```nasm
hello_world db "Hello, world!", 0
the_key     db 42
big_num     dd 0xdeadbeef
```

This generalizes to the following: `[name] d[memory size] [value]`.

You might be asking what that `[memory size]` part means. That just means the number of bytes
to allocate. Bytes in collections of sizes power of 2 have special names, such as,
* `byte` : 1 byte
* `word` : 2 bytes
* `doubleword` : 4 bytes
* `quadword` : 8 bytes

To shorten their names, doublewords and quadwords are often referred to as `dword` and `qword`
respectively.

Thus, the `dd` in the code sample above means a `dword`, or 4 bytes of memory, should
be allocated for `big_num`.

## Sections

We must now discuss sections of an executable program, as we must define these ourselves
when writing assembly code.

Sections are parts of an executable that have different purposes. They are separated for
sake of organization and permission differences when mapped into virtual memory upon
a program being run. You don't have to worry too much about what that means; just
know each section has a defined usage, and we have to put our code in each section
accordingly.

We will be using the following sections:
* `rodata` : read only memory
* `data` : read and writeable memory
* `bss` : memory allocated upon program execution
    * The Linux kernel allocates this memory when we run the process, but the
        values aren't actually stored in the executable file as in the `data`
        and `rodata` sections. You don't have to worry too much about what this
        means.
    * This section is typically used for global arrays and other large contiguous
        data structures in memory.
* `text` : actual code that is executed during runtime

You can think of the `rodata` section as storing global, constant variables;
the `data` section as storing ordinary global variables, and the `bss` section
as storing global arrays with uninitialized memory at the start of program
exeuction.

# Writing Assembly

Whew! That was a lot. You will probably have to refer to the above explanations several
times while we actually implement the assembly program, and that is perfectly fine!
Assembly is really complicated and requires a lot of low level knowledge.
If you already knew very little of the above knowledge, it will probably feel
overwhelming.

## Hello world!

We will begin by writing a simple hello world program to demonstrate
the skeleton of an assembly program.

```nasm
; soln.asm

; tell nasm that we are writing a 64bit program
bits 64

; read only data
section .rodata

; data
section .data
; Note that the 0 at the end appends a null-byte
; to terminate the string
hello_world db "Hello, world!", 0

; uninitialized memory
section .bss

; code
section .text
    global main     ; export main

; our main function
main:
    ; Push the stack pointer. Don't worry too much about this
    push rbp
    mov rbp, rsp

    ; Load the number 1 into rax. This denotes that we want to perform the
    ; WRITE system call.
    mov rax, 0x1
    ; Move the number 1 into rdi. This denotes the file we want to
    ; write to, and STDOUT has a file descriptor of 1 for (virtually)
    ; all Linux processes
    mov rdi, 0x1
    ; Load the memory address of the hello_world string into rsi
    lea rsi, [hello_world]
    ; Load the number of bytes we want to write into rdx, which is the length
    ; of our hello world string:
    mov rdx, 13
    ; Invoke the syscall
    syscall

    ; Pop the stack pointer back off and return from the main function.
    ; Don't worry too much about this.
    leave
    ret
```

This is the hello world program in assembly. A little bit longer than in Python, yeah?

You should have the knowledge required to understand most of the above assembly code,
though it will likely still be confusing. The only thing I have not explained is the
stack; that is because the stack is fairly complicated and confusing, and we won't
be making too much use of it in our code. Just know that the code at the top and
bottom of functions pertains to maintaining the stack.

## Building

We will compile this using `nasm`, `gcc`, and `make`. `make` is an old build tool
that allows us to set build targets and dependencies in a compact way. Here is
the Makefile for our project,

```make
# Our main target is the 'soln' executable
all: soln

# When we have an object file from the assembly, link it with gcc
# We also must turn off the PIE security feature for our code to link
soln: soln.o
    gcc -o $@ $? -no-pie

# Compile all assembly files into objects files with nasm, and include
# debug information in the dwarf format
%.o: %.asm
    nasm -f elf64 -g -F dwarf $<

# A target to delete all of our object files and executables
clean:
    rm *.o soln
```

We can then build the program in the command line, as well as clean
up all the object files and executables, with `make`:

```bash
$ make
$ make clean
```
# Solving the AOC Day 1 Part 1 Challenge in Assembly

Now that we have a working skeleton for our program, let us examine the
Advent of Code Day 1 Part 01 challenge:

> Specifically, they need you to find the two entries that sum to 2020 and then multiply those two numbers together.
> For example, suppose your expense report contained the following:
>
> 1721
>
> 979
>
> 366
>
> 299
>
> 675
>
> 1456
>
> In this list, the two entries that sum to 2020 are 1721 and 299. Multiplying them together produces 1721 * 299 = 514579, so the correct answer is 514579.
>
> Of course, your expense report is much larger. Find the two entries that sum to 2020; what do you get if you multiply them together?

We are given a list of numbers and told to find the pair that sum to `2020`. The answer is then
the product of those two numbers.

First, let's copy the numbers into a text file.

```txt
# input.txt

1630
1801
1917
1958
1953
1521
...
```

## Reading from the file

Our next job is to open the file and read the data into a buffer.
This will require us to first allocate a sufficiently long buffer, and then
perform the two syscalls to open the file and then read from it.

```nasm
; soln.asm

bits 64

section .rodata
    ; syscall numbers
    NR_READ        equ 0
    NR_WRITE       equ 1
    NR_OPEN        equ 2
    NR_CLOSE       equ 3

    ; file access modes
    O_RDONLY       equ 0
    O_WRONLY       equ 1
    O_RDWR         equ 2

    ; default file descriptors
    STDIN          equ 0
    STDOUT         equ 1
    STDERR         equ 2

section .data
    inputfilename  db "input.txt", 0
    inputfile      db 0
    inputbufferlen equ 1024

section .bss
    ; When we run the program, allocate inputbufferlen
    ; worth of bytes for inputbuffer
    inputbuffer    resb inputbufferlen

section .text
    global main

main:
    push rbp
    mov rbp, rsp

    ; open the input file
    mov rax, NR_OPEN            ; set our syscall to 'open'
    lea rdi, [inputfilename]    ; load the address of the filename
    mov rsi, O_RDONLY           ; set read only
    syscall

    ; if the syscall succeeded, rax will be a positive number
    ; representing the file descriptor. if it failed, it will
    ; be a negative number indicating the error.
    cmp rax, 0                  ; compare rax to 0
    jl .exit                    ; if rax < 0, there was an error, so exit

    ; read the input file
    mov [inputfile], rax        ; move the file descriptor into the global variable
    mov rdi, [inputfile]        ; load the file descriptor into rdi
    mov rax, NR_READ            ; set our syscall to 'read'
    lea rsi, [inputbuffer]      ; load the address of our buffer into rsi
    mov rdx, inputbufferlen     ; load the length of our buffer into rdx
.read:
    syscall                     ; read from the file

    ; if the read succeeded, the number of bytes read will be put into rax
    ; if it failed, rax will hold a negative number
    ; we must check for the error and also continue reading until rax holds
    ; 0, indicating that there are no bytes to read
    cmp rax, 0
    jl .cleanup                 ; if rax < 0, there was an error, so exit
    je .read_done               ; if rax == 0, we are done, so stop reading
    ; calculate the next offset in the buffer to read into and adjust the
    ; buffer length accordingly
    add rsi, rax
    sub rdx, rax
    mov rax, NR_READ
    jmp .read

.read_done:
    ; Add a zero to the end of the input from the buffer
    ; to terminate the string
    inc rsi
    mov [rsi], byte 0
    ; calculate the length of the string we read
    mov rdx, rsi
    sub rdx, inputbuffer

    ; For now, just write the buffer to stdout so we can verify this worked
    mov rax, NR_WRITE
    mov rdi, STDOUT
    lea rsi, [inputbuffer]
    syscall

.cleanup:
    ; we must now close the input file descriptor if we opened it
    ; if it is not 0, then we have opened the file and must close it
    mov rdi, [inputfile]
    cmp rdi, 0                  ; check if we opened the file
    je .exit                    ; if inputfile == 0, we did not
    ; we opened the file, so must close it using the 'close' syscall
    mov rax, NR_CLOSE           ; set our syscall to 'close'
    ; note that we already put the file descriptor into rdi
    syscall

.exit:
    leave
    ret
```

When you assemble and run this in the terminal, you should see all of the
numbers in `input.txt` be printed out to the screen.

## Converting strings to integers

Now that we have the list of numbers from the file, we have to convert the ASCII
numbers into their actual numerical values. This will be somewhat involved;
we are going to have to go byte-by-byte and extract each number into a
substring, then convert that substring to its numerical value.

It might be helpful to consult an [ASCII table](http://www.asciitable.com/)
for this step.

Let us first implement a function to take a string and convert it into a 4 byte
integer. For those who have experience with C, this is analagous to the `atoi`
function in `stdlib.h`.

As an aside, it is helpful to know the x86-64 *calling convention*. This defines
which registers represent which arguments to a function. The following are the
registers for the first four arguments, which will suffice for our purposes.
* `rdi` : 1st argument
* `rsi` : 2nd argument
* `rdx` : 3rd argument
* `rcx` : 4th argument

Additionally, when writing assembly, it is wise to write extensive documentation
for each function. Assembly is difficult to understand as it is; without extensive
documentation and comments, it is virtually impossible to parse.

```nasm
; soln.asm

; ...
section .text
    global main
    global atoi

main:
    ; ...
    leave
    ret

; Interprets a string and returns its content as an integral number.
; NOTE: the string should contain only ASCII digits and be null terminated.
; NOTE: this assumes *little endian* format.
;
; Parameters:
;   * `rdi` : pointer to the start of the string
;
; Returns:
;   * `rax` : 8 byte parsed integer
atoi:
    ; set up the stack frame
    push rbp
    mov rbp, rsp

    ; set rdx and rax to 0
    xor rax, rax
    xor rdx, rdx

; We are going to loop through all of the bytes of the string,
; turn the ASCII digit into a number from 0-9, multiply the
; existing number by 10, and then add it to the total number
;
; We will know to stop looping when the current byte is '0',
; the null byte terminating the string
;
; This total number will be stored in rax for the duration
; of the loop
.next_byte:
    mov dl, byte [rdi]          ; Get the next byte
    cmp rdx, 0                  ; If rdx is the null byte, we are done
    je .exit                    ; jump to exit
    sub rdx, 0x30               ; subtract 0x30 from the ASCII digit to map it to 0-9
    imul rax, 0xa               ; multiply the number by 10 to make space in the one's digit
    add rax, rdx                ; add the digit to the sum
    inc rdi                     ; increment our byte pointer to the next byte in the string
    jmp .next_byte              ; loop

; Upon exit, the 8 byte number is stored in rax
.exit:
    leave
    ret
```

## Parsing the input string into an integer array

Now that we can parse strings into integers, let us map the string of numbers
to an integer array. Even though our `atoi` implementation returns an 8 byte `long`,
we will treat all of the numbers as 4 byte `int`s, as they are quite small and it
saves memory.

We will need to add a new global variable for the integer array and its size, and
then implement a loop in `main` to grab a substring for each number, parse it
into an integer with `atoi`, and then add it to the array.

We will push each byte to a character array buffer, then parse it into an
integer with `atoi`. You could use push each character to the stack and
call `atoi` with a pointer to the string on the stack, but I have not
explained the stack in this post, and so we won't do that.

Remember to comment out the old debug printing code at the end of the last
code segment.

```nasm
; soln.asm

section .data
    inputfilename  db "input.txt", 0
    inputfile      db 0
    inputbufferlen equ 1024
    intarraylen    equ 200
    numberbuflen   equ 32

section .bss
    ; When we run the program, allocate inputbufferlen
    ; worth of bytes for inputbuffer
    inputbuffer    resb inputbufferlen
    ; Allocate intarraylen worth of dwords for
    ; our intarray. This allows us to store 200 integers
    intarray       resd intarraylen
    numberbuf      resb numberbuflen

section .text
    global main

main:
    ; ...

    ; Old debug print code that printed the inputbuffer to stdout
    ; comment this out for now
    ;
    ; mov rax, NR_WRITE
    ; mov rdi, STDOUT
    ; lea rsi, [inputbuffer]
    ; syscall

    ; Clear rax, rcx, rdx for the integer parsing loop
    xor rax, rax
    xor rcx, rcx
    xor rbx, rbx

    ; Load the address of inputbuffer into rsi
    lea rsi, [inputbuffer]

.int_parse:
    mov al, byte [rsi]          ; Get the next byte
    inc rsi                     ; Increment our inputbuffer string index
    cmp al, 0xa                 ; Check if the byte is a newline; if so, we are done
    je .end_of_num              ; gathering the substring

    mov [numberbuf+rcx], al     ; Load the byte into numberbuf at the rcx-th index
    inc rcx                     ; Increase our numberbuf index
    jmp .int_parse              ; Loop to next byte

.end_of_num:
    lea rdi, [numberbuf]        ; Load the address of numberbuf into rdi
    call atoi                   ; Call the atoi function on our numberbuf string
    mov [intarray+rbx*4], eax   ; Move the parsed number into the array

    ; Zero out the bytes we used in numberbuf
    xor rax, rax                ; Clear rax to 0
    lea rdi, [numberbuf]        ; move the address of numberbuf into rdi
    repe stosb                  ; Zero out the first rcx bytes of numberbuf

    ; Prepare for next loop
    inc rbx                     ; Increment our intarray index
    xor rcx, rcx                ; Clear rcx for next loop
    cmp rbx, intarraylen        ; Check if we have parsed all of the numbers
    jl .int_parse               ; If we have not parsed all the numbers, loop

    ; For now, just write the intarray to stdout to verify this works
    mov rax, NR_WRITE
    mov rdi, STDOUT
    lea rsi, [intarray]
    mov rdx, intarraylen * 4
    syscall

    ; ...
```

Make sure to comment out the old debug code.

If you assemble this code and run it, it will spit out a bunch of garbage
to your terminal and may mess up the formatting. You can fix your terminal
by pressing `Ctrl-C` and then typing `reset`. The reason that this output
is garbage is that the program is printing the raw integers to `stdout`;
it is *not printing in ASCII*. Your terminal attempts to print these bytes,
but they just translate to weird symbols.

To check that the output is correct, pipe the program output to a file
and then examine the hexdump as so:

```bash
$ ./soln > bytes
$ hd bytes
00000000  5e 06 00 00 09 07 00 00  7d 07 00 00 a6 07 00 00  |^.......}.......|
00000010  a1 07 00 00 f1 05 00 00  c6 07 00 00 a7 07 00 00  |................|
00000020  07 06 00 00 06 07 00 00  7e 02 00 00 db 05 00 00  |........~.......|
00000030  b9 07 00 00 99 05 00 00  fc 05 00 00 f4 06 00 00  |................|
00000040  17 06 00 00 4a 07 00 00  aa 07 00 00 cf 07 00 00  |....J...........|
00000050  57 06 00 00 ec 06 00 00  c2 06 00 00 86 06 00 00  |W...............|
00000060  ff 06 00 00 9b 07 00 00  a9 07 00 00 f3 05 00 00  |................|
00000070  bf 03 00 00 ce 07 00 00  9d 06 00 00 d2 05 00 00  |................|
# ...
$ python -c "print 0x065e"
1630
$ python -c "print 0x05d2"
1490
```

Every 4 bytes represents a 4 byte number in little endian format. That is,
the first 4 bytes `5e 06 00 00` translates to the 4 byte number `0x0000065e`.
We can use Python to print these numbers are in decimal and compare them
to the numbers in the `input.txt` file. You can verify that these two
numbers are the 1st and 32nd numbers in the `input.txt` file, respectively.
Feel free to check that more numbers match if you like.

## Finding two numbers that sum to 2020

There are a number of clever ways to find two numbers in the integer array
that sum to 2020, but for simplicity's sake, we will just bruteforce it with
the O(n^2) algorithm and try every possible combination.

To do this, we will have to
1. maintain two indices into `intarray` to get pairs of numbers
2. loop through indices starting from the first index to the last index
3. increment the first index and repeat until we find a pair that sum to 2020
4. exit out of the loop, print the numbers, and print their sum.

To print their sum, it will be easiest to use the `printf` function from
`libc`. Otherwise, we would have to reconvert our numbers into ASCII or
recalculate their initial substring index in our `inputbuffer`. Luckily,
`gcc` will automatically link against `libc`, so we merely have to add
the format string to our `.data` section and write `extern printf` to
import the function.

Remember to comment out the old debug printing code from before.

```nasm
; soln.asm

section .data
    ; ...
    foundpairstr   db "Found pair: %d * %d = %d", 10, 0

; ...

section .text
    extern printf
    global main

main:
    ; ...

    ; For now, just write the intarray to stdout to verify this works
    ; mov rax, NR_WRITE
    ; mov rdi, STDOUT
    ; lea rsi, [intarray]
    ; mov rdx, intarraylen * 4
    ; syscall

    ; Our starting indices into intarray, held in r12 and r13
    mov r12, 0
    mov r13, 1
.find_pair:
    mov eax, [intarray+r12*4]   ; Move the first 4 byte number into eax
    add eax, [intarray+r13*4]   ; Add the second number to get the sum
    cmp eax, 2020               ; Check if their sum is 2020
    je .found_pair              ; If so, exit the loop
    inc r13                     ; Increment the inner loop variable
    cmp r13, intarraylen        ; Check if we just reached the last number
    jne .find_pair              ; If not, loop
    inc r12                     ; If so, increment the outer loop variable
    mov r13, r12                ; and set the inner loop variable to be one
    inc r13                     ; one more
    jmp .find_pair              ; Loop
.found_pair:
    ; Clear the following registers in case there is
    ; data in the top 32 bits that will change the
    ; numbers we will print
    xor rsi, rsi
    xor rdx, rdx
    xor rcx, rcx
    ; We have found the pair, so print them and their product out
    lea rdi, [foundpairstr]     ; Load the address of the string into rdi
    mov esi, [intarray+r12*4]   ; Load the first 4 byte number into esi
    mov edx, [intarray+r13*4]   ; Load the second 4 byte number into edx
    mov ecx, esi                ; Move the first number into ecx
    imul ecx, edx               ; Multiply by edx to get their product
    xor rax, rax                ; Clear rax (required by printf)
    call printf                 ; Call printf
```

Note that we have to multiply the indices stored in `r12` and `r13` by 4 to
get their byte offset in the integer array. Also note that when moving a 4
byte integer to a register, we must use the 4 byte version of the register;
such as `eax`, `esi`, `edx`, and `ecx` above. Furthermore, when doing this,
we must make sure that the upper 4 bytes of the register do not impact our
results. We do not clear `rax` before our loop because we only ever work
with `eax` and thus never access the upper 4 bytes; however, in our printing
code, we must clear the 64bit registers before moving the numbers into their
lower 4 bytes so that the numbers we pass to `printf` are correct.

# Conclusion

If you followed along with me, you now have working code to solve the first part
of the Day 1 AOC challenge. I will leave the second part as an exercise; the code
is very similar to that of part 1, except that the brute force solution has three
nested loops instead of two. Because it is so similar, I highly recommend that you
attempt to complete it yourself as an exercise. If you get stumped or just want the
solution, you can see my solution [here](https://github.com/alidiusk/aoc2020-asm/blob/master/day01/soln.asm).

If you are interested in learning more assembly, the following resources were
incredibly helpful for me in learning the assembly and low level knowledge
required for this article,
1. Assembly Language Step by Step with Linux, Third Edition, by Jeff Duntemann
2. Beginnning x86 Assembly Programming: From Novice to AVX Professional, by
Joe Van Hoey
3. The Linux Man Pages

If you have any questions, problems, or feedback while following this
article, feel free to create an  [issue](https://github.com/alidiusk/aoc2020-asm/issues)
on my [solutions repository](https://github.com/alidiusk/aoc2020-asm) or
[email me](emailto:liamowoodward@gmail.com) directly.
