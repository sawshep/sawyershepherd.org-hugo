---
title: "Hack The Box \"Delivery\" Writeup"
date: 2023-09-25T22:39:40-04:00
draft: false
---

[Delivery](https://app.hackthebox.com/machines/Delivery) is
a retired easy Linux box by ippsec, available on Hack The
Box. It was released on January 9th, 2021.

## Scanning

I began by scanning the machine:
```bash
$ nmap 10.10.10.222 -p- -O
...
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8065/tcp open  unknown
...unknown-linux-gnu...
```

Let's run a more intensive scan on the open ports:
```bash
$ nmap -p 22,80,8065 -sV -sC -T4
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 9c40fa859b01acac0ebc0c19518aee27 (RSA)
|   256 5a0cc03b9b76552e6ec4f4b95d761709 (ECDSA)
|_  256 b79df7489da2f27630fd42d3353a808c (ED25519)
80/tcp   open  http    nginx 1.14.2
|_http-server-header: nginx/1.14.2
|_http-title: Welcome
8065/tcp open  unknown
...<title>Mattermost</title>
```

We have and SSH server on port 22, a webserver on port 80,
and a Mattermost server on port 8086 (also a web) server.

## Enumerating SSH

There are no publicly available exploits for OpenSSH 7.9p1
at the time of this writing. Password authentication is
enabled on the server.

## Enuerating NGINX

Let's connect to the webserver in a browser:

![](/images/hack-the-box-delivery-writeup/home.jpg)

It's a pretty standard framework homepage. Let's check out
the links at the bottom of the page. One leads to
`delivery.htb/#contact`, and the other
`helpdesk.delivery.htb`. The contact page is similar, though
we're given some additional interesting information:

> For unregistered users, please use our HelpDesk to get in
> touch with our team. Once you have an @delivery.htb email
> address, you'll be able to have access to our MatterMost
> server.

This confirms that Mattermost is running on port 8065. Let's
check out the helpdesk: 

![](/images/hack-the-box-delivery-writeup/submit-ticket.jpg)

After submitting a ticket, we are presented with a receipt,
where we are given a ticket number (1769678) and an email
to which we can send further information
(1769678@delivery.htb). We can also check our ticket status
on the "Check Ticket Status" page:

![](/images/hack-the-box-delivery-writeup/ticket-status.jpg)

## Enumerating Mattermost

Now that we have an email, we can log in to Mattermost,
according to the website's contact page,
`http://delivery.htb:8065`. I tried a few passwords using
"1769678@delivery.htb" as the username, but none of them
worked. Next, I tried creating an account with the email
instead. Sure enough, it was accepted, but then the
registration flow prompted me to confirm the email:

![](/images/hack-the-box-delivery-writeup/registration.jpg)

Remember that we used the ticked-provided email to register.
Let's check the ticket status page and see if we can access
the confirmation email there:

![](/images/hack-the-box-delivery-writeup/email.jpg)

After following the activation link, we can now log in to
the server...

![](/images/hack-the-box-delivery-writeup/verified.jpg)

## Pwning User

...and view the internal communications!

> root
> 2:29 PM
> @developers Please update theme to the OSTicket before we
> go live.  Credentials to the server are
> maildeliverer:Youve_G0t_Mail! 
> Also please create a program to help us stop re-using the
> same passwords everywhere.... Especially those that are a
> variant of "PleaseSubscribe!"

> root
> 3:58 PM
> PleaseSubscribe! may not be in RockYou but if any hacker
> manages to get our hashes, they can use hashcat rules to
> easily crack all variations of common words or phrases.

We're given a username and password combo, and some clues
about how to proceed with the box. You'll see later I
ignored these clues and exploited a different vulnerability.

We can log in to the server with the provided username and
password:

```
$ ssh maildeliverer@delivery.htb
maildeliverer@delivery.htb's password: Youve_G0t_Mail!
Linux Delivery 4.19.0-13-amd64 #1 SMP Debian 4.19.160-2 (2020-11-28) x86_64
...
maildeliverer@Delivery:~$ 
```

Fantastic! We can now retrieve the flag from `user.txt`.

## Pwning Root

Let's run some privilege escalation scripts.
[PEASS-ng](https://github.com/carlospolop/PEASS-ng) is very
popular, but I prefer
[LSE](https://github.com/diego-treitos/linux-smart-enumeration).
I cloned the repository to my machine, built in CVE scanning
with the provided tool, and used SCP to transfer the script
to the target box.

I ran the script and collected the most interesting
findings:
```
[!] sof080 Can we write to a gpg-agent socket?........... yes!
[!] cve-2021-4034 Checking for PwnKit vulnerability...... yes!
Vulnerable! polkit version: 0.105-25
[!] cve-2023-22809 Sudoedit bypass in Sudo <= 1.9.12p1... yes!
Vulnerable! sudo version: 1.8.27-1+deb10u3
```

I first looked into the `sudoedit` bypass, but as the
`maildeliverer` user was not a sudoer, this exploit would
not work.

PwnKit, however, proved to be fruitful. I downloaded and
built a self contained exploit from the [PwnKit
repository](https://github.com/ly4k/PwnKitK) and transferred
it to the target via SCP. I ran it on the target and I was
escalated to root without a hitch!

PwnKit works by exploiting a flaw of the PolKit pkexec
program, in which the number of arguments is not validated,
resulting in execution of environment variables[^pwnkit].

With root access, I could finally read `root.txt` and finish
the box.

[^pwnkit]: RHSB-2022-001 Polkit Privilege Escalation -
  (CVE-2021-4034),
  [https://access.redhat.com/security/vulnerabilities/RHSB-2022-001](https://access.redhat.com/security/vulnerabilities/RHSB-2022-001)

