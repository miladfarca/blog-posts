#### Objective
In this blog we are going to see how an executable binary file can be edited using native tools on most Linux distributions.

#### What you need
```
vim (or any other text editor)
gcc (or any other C compiler)
xxd
objdump
```

#### Procedure
Let's first create a simple executable binary file which prints the number `123` on the screen.
This program will be written in C, open a new file in your text editor, name it `print.c` and paste the following:
```c
#include <stdio.h>

int main(){
 printf("%d\n", 123);
 return 0;
}
```
Compile it with `gcc print.c -o print` which outputs an executable file called `print`.
Running `./print` should display `123` on your terminal:
```text
# ./print
123
```

Now let's use `objdump` to examine the instructions within this binary file:
```
# objdump -d print
...
0000000000001149 <main>:
    1149:	f3 0f 1e fa          	endbr64 
    114d:	55                   	push   %rbp
    114e:	48 89 e5             	mov    %rsp,%rbp
    1151:	be 7b 00 00 00       	mov    $0x7b,%esi
    1156:	48 8d 3d a7 0e 00 00 	lea    0xea7(%rip),%rdi        # 2004 <_IO_stdin_used+0x4>
    115d:	b8 00 00 00 00       	mov    $0x0,%eax
    1162:	e8 e9 fe ff ff       	callq  1050 <printf@plt>
    1167:	b8 00 00 00 00       	mov    $0x0,%eax
    116c:	5d                   	pop    %rbp
    116d:	c3                   	retq   
    116e:	66 90                	xchg   %ax,%ax
...
```
Note that there are many additional printed lines which we are interested in at the moment, we will only be examining the output of the `main` function.

The number `123` is displayed as `0x7b` in hexadecimal format. In the above generated instructions `0x7b` is being put into the `esi` register
before calling `printf`.

Let's use `xxd` to get the hexdump of the file and save it separately:
```
# xxd print > print.hex
```
The hexdump has more than 1000 lines of content. We ware going to search for the `7b` pattern and change it to `7c` which represents `124` in decimal.

The line will look like this before changing:
```
00001150: e5be 7b00 0000 488d 3da7 0e00 00b8 0000  ..{...H.=.......
```

Simply change `7b` to `7c` and save:
```
00001150: e5be 7c00 0000 488d 3da7 0e00 00b8 0000  ..{...H.=.......
```

Note that `endianness` is not a concern here as we are only changing a single byte. We will talk about this in more detail in a future post.

Now we are going to reuse `xxd` to convert this hexdump back into binary, we will need to ue the `-r` flag for reverse operation.
```
# xxd -r print.hex > ./print_changed 
```

The newly generated binary file is not executable, we will need to mark it as such using `chmod`:
```
# chmod +x ./print_changed
```

Now you can run this new binary file and see the output has changed from `123` to `124`:
```
# ./print_changed 
124
```

### Why can't we just read/write in binary and avoid xxd?
You can't directly write/type in binary using your keyboard in a regular text editor. When you write `0` you are representing the `0` character and not the `0` integer.
A character is 1 byte (8 bits). The byte sized integer `0` is saved as `00000000` in memory or other media as binary, however the character `0` is
saved as `110000`. The list of characters and their binary and hex representations can be seen in an `Ascii Table`.
You will need a converter such as `xxd` to convert your characters to binary and vice versa. This is why binary files are not generally readable using a text editor. Trying to 
do so will print gibberish on your screen similar to this:
```
^B^A^A^@^@^@^@^@^@^@^@^@^C^@>
```
Note there are many `hex editors` which will do the conversion for you behind the scenes. We have used `xxd` here as its commonly available in many
Linux distros.

We will be writing our own simple converter in a future post.
