

#### Objective
In this blog we are going to build a small arm64 emulator and use it to emulate a function.
We will also measure and compare the runtime performance of emulating vs running it natively.

#### What you need
```text
gcc
gcc-aarch64-linux-gnu 
binutils-aarch64-linux-gnu
```
### Source code
Source code can be found in this repository: [https://github.com/miladfarca/arm64-emulator](https://github.com/miladfarca/arm64-emulator)

#### Procedure
Emulators usually load an external binary file and emulate its instructions. To make the process easier, the binary
we are going to emulate is part of the emulator binary itself.

The process can be divided into the following steps:

1- Using the arm64 compiler toolchain, compile the following function which adds two integers:
```c
int add(int a, int b)
{
    return a + b;
}
```
The following command creates an object file named `add.o` containing arm64 instructions:
```text
# aarch64-linux-gnu-gcc -c add.c
``` 

2- Disassemble the generated object file to see the instructions in hexadecimal format:
```text
# aarch64-linux-gnu-objdump -d add.o

0000000000000000 <add>:
   0:	d10043ff 	sub	sp, sp, #0x10
   4:	b9000fe0 	str	w0, [sp, #12]
   8:	b9000be1 	str	w1, [sp, #8]
   c:	b9400fe1 	ldr	w1, [sp, #12]
  10:	b9400be0 	ldr	w0, [sp, #8]
  14:	0b000020 	add	w0, w1, w0
  18:	910043ff 	add	sp, sp, #0x10
  1c:	d65f03c0 	ret
```
Note: Instructions are stored in `little endian` order in memory, they are displayed in human readable order with `objdump` above.
As an example, `sub	sp, sp, #0x10` is stored as `0xff 0x43 0x00 0xd1`.

3- Extract the hex values and store them in a separate assembly file (`.s` file) using `.long` directives. We need to compile our emulator source code with the `add` function included in the final binary. The source file may be compiled in a non arm64 environment (i.e `x64`), however we still need to make sure the `add` function is compiled into `arm64` instructions. We cannot use native `arm64` opcodes in the assembly file as the `x64` assembler cannot understand them. We need to extract their hexadecimal values and emit them as integer values which is understandable by most assemblers.

We also need to make sure `arm64_add` is marked as global using the `.globl` directive so the linker can find it later in the process.

4- The emulator will point to the first byte of `arm64_add` function. The pointer is `32 bit integer` sized. We load 32 bit values into memory and examine them. If the value matches our list of instructions then we will emulate them according to the `aarch64 instruction set architecture`.

### The emulator
In order to execute the `add` function, we need to emulate the following 6 instructions of arm64:
```text
ldr: Load
str: Store
add: Add register, immediate
sub: Subtract register, immediate
add: Add register, register
ret: Return
```
The emulator needs to follow the `arm64 ABI`. Parameters are initially placed into registers `w0` and `w1`. 
The stack pointer needs to be initialized and its value must be placed into the stack pointer register `w31`.

#### Runtime performance of emulating vs running on a native host
The emulator contains a loop to run the `add` function natively as well as using the emulator which helps us 
compare their runtime performance. The loop runs `33,554,432` times.

We will compile and run both versions on the same arm64 host, the machine details are as follows:
```text
Architecture:        aarch64
Model name:          Cortex-A53
CPU(s):              6
RAM:                 2 Gb
```

**Native run**
```text
# time ./emualte_arm64 --test-native

real	0m0.353s
user	0m0.300s
sys	0m0.012s
```

**Emulator**
```text
# time ./emualte_arm64 --test-emulator

real	0m35.218s
user	0m23.372s
sys	0m0.128s
```

Tests were executed multiple times and results were similar in value. This clearly shows how emulation is **significantly** slower than running instructions natively. The following figure shows the data using a bar graph:

<image width="70%" src="https://github.com/miladfarca/blog-posts/raw/main/images/2-a.png">
