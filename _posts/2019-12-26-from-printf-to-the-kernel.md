---
layout: post
title:  "From printf() to the kernel"
date:   2019-12-26 11:04:47 +0000
categories: tutorial
---

Consider:

```c
#include <stdio.h>

int main(int argc, char **argv) {
	printf("Hello world\n");
	return 0;
}
```

Compiling and running:

```
# gcc main.c
# ./a.out 
Hello world
```

What's actually happening to get from the `printf` line to the characters on the screen?

printf is a call into libc's printf function. To make it a bit simpler recompile with `-static` so we don't have dynamic linking and look at the disassembly.

```
# gcc -static -g main.c
# objdump -D a.out | less
```

```
00000000004009ae <main>:
...
  4009b6:       89 7d fc                mov    %edi,-0x4(%rbp)
  4009b9:       48 89 75 f0             mov    %rsi,-0x10(%rbp)
  4009bd:       bf 24 11 4a 00          mov    $0x4a1124,%edi
  4009c2:       e8 19 f2 00 00          callq  40fbe0 <_IO_puts>
...
```

The compiler is smart enough to see what we're not doing any dynamic string generation with printf, so it directly calls [_IO_puts](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob_plain;f=libio/ioputs.c;hb=HEAD) which is mostly a light wrapper around `IO_sputn` which eventually makes its way to a `write` syscall on x86.

```
# strace ./a.out 
...
write(1, "Hello world\n", 12Hello world
)           = 12
...
```

We can see that the output is printed between the call and the return.

We can remove the libc abstraction for now by doing the syscall directly.

```
#include <unistd.h>

int main(int argc, char **argv) {
	char buf[] = "Hello world\n";
	// ssize_t write(int fd, const void *buf, size_t count);
	write(2, buf, sizeof(buf));
	return 0;
}
```

But it turns out that's a libc wrapper function as well. Going to assembly to do the syscall directly.

```
# cat asm.s
.intel_syntax noprefix
.text
.globl _start 
_start:
  mov eax, 1 # syscall number
  mov edi, 1 # fd = 1 = stdout
  lea esi, my_string # buf
  mov edx, 12 # len
  syscall
fin:
  jmp fin

.data
my_string:
.string "Hello World\n"

# as asm.s -o asm.a
# ld asm.a -o asm
# objdump -D asm 
...
00000000004000b0 <_start>:
  4000b0:	b8 01 00 00 00       	mov    $0x1,%eax
  4000b5:	bf 01 00 00 00       	mov    $0x1,%edi
  4000ba:	8d 34 25 ca 00 60 00 	lea    0x6000ca,%esi
  4000c1:	ba 0c 00 00 00       	mov    $0xc,%edx
  4000c6:	0f 05                	syscall 

```

Where 0x6000ca is the address of the string. Sure enough, running `./asm` produces the expcted output and we can see the syscall in action with strace.

We've passed in 1 for the write file descriptor. This fd was set up for us for.

## Buffering

If we print out some lines without newline characters:

```
#include <stdio.h>
#include <unistd.h>

int main(void) {
  printf("Hello world");
  sleep(1);
  printf("Hello world");
  sleep(1);
  printf("Hello world");
  sleep(1);
  return 0;
}
```
They don't get printed immediately after the printf; this output appears all at once:

```
./a.out 
Hello worldHello worldHello world%  
```

And it's a single syscall:

```
strace ./a.out
...
nanosleep({1, 0}, 0x7ffe19052230)       = 0
nanosleep({1, 0}, 0x7ffe19052230)       = 0
nanosleep({1, 0}, 0x7ffe19052230)       = 0
write(1, "Hello worldHello worldHello worl"..., 33Hello worldHello worldHello world) = 33
```

So clearly printf is buffering the syscall. It flushes the buffer with a `write()` once it gets a new line character or when the program exits.

How does it flush just before exit?

```
0000000000400890 <_start>:
  400890:       31 ed                   xor    %ebp,%ebp
  400892:       49 89 d1                mov    %rdx,%r9
  400895:       5e                      pop    %rsi
  400896:       48 89 e2                mov    %rsp,%rdx
  400899:       48 83 e4 f0             and    $0xfffffffffffffff0,%rsp
  40089d:       50                      push   %rax
  40089e:       54                      push   %rsp
  40089f:       49 c7 c0 50 17 40 00    mov    $0x401750,%r8
  4008a6:       48 c7 c1 c0 16 40 00    mov    $0x4016c0,%rcx
  4008ad:       48 c7 c7 ae 09 40 00    mov    $0x4009ae,%rdi  # this is the address of main
  4008b4:       e8 47 09 00 00          callq  401200 <__libc_start_main>
```

`_start` -> `__libc_start_main`  -> `generic_start_main` -> `main`

Once C's `main()` function returns, flow goes back to `generic_start_main` and the exist sequence runs. Just before it's about to do the syscall instruction:

```
(gdb) backtrace
#0  0x000000000043f4be in __write_nocancel ()
#1  0x000000000041263f in _IO_new_file_write ()
#2  0x0000000000413469 in _IO_new_do_write ()
#3  0x000000000041444a in _IO_cleanup ()
#4  0x000000000040ead8 in __run_exit_handlers ()
#5  0x000000000040eb83 in exit ()
#6  0x0000000000400c4d in generic_start_main ()
#7  0x0000000000401235 in __libc_start_main ()
#8  0x00000000004008b9 in _start ()
```

Where's the buffer?

```
(gdb) info reg esi
esi            0x6cf880 7141504

(gdb) x/s 0x6cf880
0x6cf880:       "Hello worldHello worldHello world"
```

Which is some dynamic space that libc must have malloc'd:

```
# pmap -x $(pidof a.out)          
812:   /tmp/scratch/a.out
Address           Kbytes     RSS   Dirty Mode  Mapping
00000000006cc000     148      16      16 rw---   [ anon ]
```


## References

https://jameshfisher.com/2018/02/19/how-to-syscall-in-c/

https://stackoverflow.com/questions/2535989/what-are-the-calling-conventions-for-unix-linux-system-calls-on-i386-and-x86-6
