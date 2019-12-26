---
layout: post
title:  "How Stdout works"
date:   2019-12-26 11:04:47 +0000
categories: jekyll update
---

## Users space

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

printf is a call into libc's printf function. Let's pretend that dynamic linking doesn't exist for now, so recompile with `-static` and `-g` to keep symbols, and look at the disassembly.

```
# gcc -static main.c
# objdump -D a.out | less
```

```
00000000004009ae <main>:
  4009ae:       55                      push   %rbp
  4009af:       48 89 e5                mov    %rsp,%rbp
  4009b2:       48 83 ec 10             sub    $0x10,%rsp
  4009b6:       89 7d fc                mov    %edi,-0x4(%rbp)
  4009b9:       48 89 75 f0             mov    %rsi,-0x10(%rbp)
  4009bd:       bf 24 11 4a 00          mov    $0x4a1124,%edi
  4009c2:       e8 19 f2 00 00          callq  40fbe0 <_IO_puts>
  4009c7:       b8 00 00 00 00          mov    $0x0,%eax
  4009cc:       c9                      leaveq 
  4009cd:       c3                      retq   
  4009ce:       66 90                   xchg   %ax,%ax
```

The compiler is smart enough to see what we're not doing any dynamic string generation with printf, so it directly calls [_IO_puts](https://sourceware.org/git/gitweb.cgi?p=glibc.git;a=blob_plain;f=libio/ioputs.c;hb=HEAD) which is mostly a light wrapper around `IO_sputn` which eventually makes its way to a `write` syscall on x86.

```
# strace ./a.out 
...
write(1, "Hello world\n", 12Hello world
)           = 12
exit_group(0)                           = ?
+++ exited with 0 +++
```

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
  mov eax,1 # syscall number
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

## References

https://jameshfisher.com/2018/02/19/how-to-syscall-in-c/

https://stackoverflow.com/questions/2535989/what-are-the-calling-conventions-for-unix-linux-system-calls-on-i386-and-x86-6
