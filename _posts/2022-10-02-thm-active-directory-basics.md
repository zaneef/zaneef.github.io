---
title: THM - Active Directory Basics
tags: [thm, windows, active directory]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/3f520838881aee0c6a245ed2d35bb9dc.png" alt="Challenge logo" style="height:150px;">
</div>

## Introduction

Microsoft's Active Directory is the backbone of the corporate world. It simplifies the management of devices and users within a corporate envireoment. In this room, we'll take  adeep dive into the essential components of Active Directory.

## Windows Domains

Picture yourself administering a small business network with only five computers and five employees. In such a tiny network, you will probably be able to configure each computer separately without a problem. Now imagine your business suddently grow and now haw 157 computers and 320 different users across four different offices. You won't probably be able to configure each computer and provide on-site support for everyone.

To overcome these limitation, we can use a Windows Domain. A Windows Domain is a group of users and computers under the administration of a given business. The main idea is to centralise the administration of common components of a Windows computer network in a single repository called Active Directory (AD). The server that runs the Active Directory services is known as a Domain Controller (DC).

During this task, we'll assume the role of the new IT admin at THM Inc. As our first task, we have been asked to review the current domain `THM.local` and do some additional configurations. You will have administrative credentials over a pre-configured Domain Controller (DC) to do the tasks.

We have to start the machine and log into it through RDP.

## Active Directory

The core of any Windows Domain is the Active Directory Domain Service (AD DS). This service acts as a catalogue that holds the information of all of "objects" that exist on your network. 

**Users**

Users are one of the most common object types in Active Directory. Users are one of the objects known as security principals, meaning that they can be authenticated by the domain and can be assigned privileges over resources like files or printers. Users can be used to represent two types of entities:

- People: users will generally represent persons in your organisation that need to access the network, like employees
- Services: youcan also define users to be used by services like IIS or MySQL.

**Machines**

For every computer that joins the Active Directory domain, a machine object will be created. Machines are also considererd "security principals" and are assigned an account just as regular user. 

The machine accounts themselves are local administrators on the assigned computer. As with any other account, if you have the password, you can use it to log in.

Identyfing machine accounts is relatively easy. Thay follow a specific naming scheme. The machine account name is the computer's name followed by a dollar sign.

**Security Groups**

You can define user groups to assign access rights to files or other resources to entire groups instead of single users. This allows for better manageability as you can add users to an existing group, and they will automatically inherit all of the groups privileges.

**Active Directory Users and Computers**

To configure users, groups or machines in Active Directory, we need to log in to the Domain Controlelr an type "Active Directory Users and Computers" from the start menu.

This will open up a window where you can see the hierachy of users, computers and groups that exist in the domain. These objects are organised in Organizational Units (OUs) which are container objects that allow you to classify users and machines.

## Managing Users in AD

We have to delete the "Research and Development" Organizational Unit. We can't simply remove it since OUs are protected by default against accidental deletion. We have to click on the "View" tab and click on "Advanced Features". Right click over the OU and go to "Properties", go to the "Object" tab and disable "Protext object from accidental deletion". We can now delete the OU.

**Delegation**

One of the nice things you can do in AD is to give specific users some control over some OUs. This process is known as **delegation** and allows you to grant user specific privileges to perfom dvanced tasks on OUs without needing a Domain Administrator to step in.

According to our organisational chart, Phillip is in charge of IT support, so we'd probably want to delegate the control of resetting passwords to him.

Right click on one OU (Sales for example) and got to "Delegate Control". Click "Add", type "phillip" on the text area and click "Check Names": this should autocomplete che user for you.

Click "Ok" and "Next" a few times. Now Phillis should be able to reset password on users on the Sales department. Log into Phillip account using the credentials given on the box and open a PowerShell instance.

The following command would change the password for Sophie's user. 

```ps1
PS C:\Users\Phillip> Set-ADAccountPassword sophie -Reset -NewPassword (Read-Host -AsSecureString -Prompt 'New Password') -Verbose

New Password: ********
```

Now log into the machine and get the flag on Sophie's desktop.

Since we don't want to know Sophie's password, we can force a password reset at the next logon with the following command

```ps1
PS C:\Users\Phillip> Set-ADUser -ChangePasswordAtLogon $true -Identity sophie -Verbose
```

## Managing Computers in AD

By default, all the machines that join a domain (except for DCs) will be put in the container called "Computers". 

It's not recommended to have all types of computers on the same OU. Let's create two new OUs called "Workstations" and "Servers" and move the computers in the respective OU.

We have now 7 computers on the "Workstations" OU (Laptop + PC) and 3 on the "Servers" one.

## Group Policies

So far we have organised users and computers in OUs just for the sake of it, but the main idea behind this is to be able to deploy different policies for each OU individually.

Windows manages such policies through Group Policy Objects (GPO). GPOs are simply a collection of settings that can be applied to OUs.

To configure GPOs, you can use the Group Policy Management tool, available from the start menu. To configure Group Policies, you first create a GPO under Group Policy Objects and then link it ot the GPO where you want the policies to apply.

**Restrict Access to Control Panel**

We want to restrict access to the Control Panel across all the machines to only the users that are part of the IT department. Users of other departments shouldn't be able to change the system's preferences.

Create a new GPO called "Restrict Control Panel Access" and open it for editing. Since we want this GPO to apply to specific users, we will look under "User Configuration" for the following policy. Go to "Policies > Administrative Templates" and click on "Control Panel". Click on "Prohibit access to Control Panel and PC settings" and enable it.

Once the GPO is configured, we can link it to all the OUs corresponding to users who shouldn't have access to the Control Panel of their PCs. Drag the policy to the corresponding OUs.

**Auto Lock Screen GPO**

We want to autolock the screen of all PCs after 5 minutes of inactivity. Create a new policy and edit it. Go to "Computer Configuration > Policies > Windows Settings > Security Settings" and search "Interactive logon: Machine inactivity limit". Change its setting to 300 seconds.

**GPO distribution**

GPOs are distributed to the network via a network share called SYSVOL, which is stored in the DC. All users ina a domain should tipically have access to this share over the network to sync their GPOs periodically. The SYSVOL share points by default to the C:\Windows\SYSVOL\sysvol directory on each of the DCs in our network.

Once a change has been made to any GPOs, it might take up to 2 hours for computers to catch up. If you want to force it type the following command on the desired computer

```ps1
PS C:\> gdupdate /force
```

## Authentication Methods

When using Windows domains, all credentials are stored in the Domain Controllers. Whenever a user tries to authenticate to a service using domain credentials, the service will need to ask the Domain Controller to verify if they are correct. Two protocols can be used for network authentication in windows domains:

- Kerberos: used by any recent version of Windows
- NetNTLM:  legacy authentication protocol kept for compatibility purpose

**Kerberos Authentication**

Users who log into a service using Kerberos will be assigned tickets. Think of tickets as proof of a previous authentication.

When Kerberos is used for authentication, the following process happens:

- The user sends their username and a timpestamp encrypted using a key derived from their password to the Key Distribution Center (KDC), a service usually installed on the Domain Controller in charge of creating Kerberos tickets on the network. The KDC will create a Ticket Granting Ticket (TGT), which allow the user to request additional tickets to access specific services. This prevents the user's credentials to be sent over the network everytime he wants to connect to a service. Along with the TGT, a Session Key is given to the user.
- When a user want to connect to a service on the network like a share, website or database, they will use their TGT to ask the KDC fot a Ticket Granting Service (TGS). To request a TGS, the user will send their username and a timestamp ancrypted using the Session Key, along with the TGT and a Service Principal Name (SPN), which indicates the service and server name we intend to access. The KDC sens us a TGS along with a Service Session Key, which we will need to authenticate to the service we want to access. The TGS is encrypted using a key derived from the Service Owner Hash. 
- The TGS can be sent to the desired service to authenticate and establish a connection.