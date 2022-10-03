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