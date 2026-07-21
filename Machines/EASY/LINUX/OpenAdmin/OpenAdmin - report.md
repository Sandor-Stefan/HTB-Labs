
- I start with an IP address and first thing I start a `nmap` scan to enumerate the available ports
```shell
nmap -sC -sV 10.10.10.171
```
![[Pasted image 20250830194147.png]]
- There are 2 ports open:
	- port `22` open for `ssh`
	- port `80` open for HTTP
- On port `80` there is a web application, with which I am starting the exploitation process

- The site is powered by `Apache` which I can also see on the main page of the web app
![[Pasted image 20250830195034.png]]

- Next I start an `ffuf` scan to fuzz for available directories 
```shell
ffuf -u http://10.10.10.171/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic -t 150 
```
- The scan reveals some directories, nothing interesting, but it is a starting point
![[Pasted image 20250830195758.png]]

- The first directory I check is the `/music` one which leads me to music software page
![[Pasted image 20250830200145.png]]

- Clicking the `Login` button from the `MENU` opens up a whole another page
![[Pasted image 20250830202041.png]]
- The page that opens up is found at the `/ona` and it is an `OpenNetAdmin` page
![[Pasted image 20250830202532.png]]

- We can also see the version of `OpenNetAdmin` which is `v18.1.1`
- Searching the Internet for some information about this version I got a vulnerability, more exactly the [CVE-2019-25065](https://nvd.nist.gov/vuln/detail/cve-2019-25065), which leads to OS commands Injection
- I also found an exploit for this vulnerability with the help of `searchsploit`
![[Pasted image 20250830203847.png]]
- Now the only thing that is left is to try it
```shell
./47691.sh http://10.10.10.171/ona/login.php
```
- The exploit worked and it lent me a reverse shell as the `www-data` user
![[Pasted image 20250830204024.png]]
- Now that I have access to the web app directory I searched through it until I found some credentials in the `/opt/ona/www/local/config/database_settings.inc.php` 
![[Pasted image 20250830204717.png]]
- These credentials are from the  `OpenNetAdmin` app database, but the most interesting one seems the password `n1nj4W4rri0R!` which seems changed while all the other are default things

- I had a moment of thought and I was thinking that maybe this password belongs to one of the users, so next thing to check is the `/home` directory to see what users we have there
![[Pasted image 20250830205014.png]]
- The `/home` directory contains 2 users directories:
	- `jimmy`
	- `joanna` 
- Now that I have 2 usernames and a password I can try and maybe successfully to `ssh` into one of the 2 users accounts

- First I try with the `jimmy` user and the password `n1nj4W4rri0R!`, and look at that it worked and now I have access as the `jimmy user`
![[Pasted image 20250830205310.png]]
- Searching for the flag inside the `jimmy` home directory, I was dumbfounded to find out that there is nothing besides the basic files and directories, so no `user.txt` flag which means the flag must be inside the `joanna` home directory
![[Pasted image 20250830205534.png]]

- Trying to check for the flag inside `joanna`'s home directory I was met with the most annoying prompt `Permission denied` and when checking her directory permissions I learned that only she has access to see the contents of the directory
![[Pasted image 20250830205701.png]]

- Next on the list is to check if the `n1nj4W4rri0R!` password works also for `joanna` account
- Unfortunately when trying the same password for the `joanna` account, it failed meaning that I need to search again through the target server for her password

- Now trying to go again to the web app directory I stumbled across another directory inside the `/var/www` one
![[Pasted image 20250830211207.png]]
- Searching through it, I found 3 other `php` scripts/web pages, but one of them, `main.php` had a gold mine in it 
![[Pasted image 20250830211505.png]]
- A `shell_exec` command that could execute shell commands and especially to output the content of the `joanna`'s `id_rsa` file which contains the key to connect to server through `ssh` as the `joanna` user
- So next step is to find a way to make either the `root` or `joanna` to execute the script

- It also got me thinking that if this directory is inside the `/var/www` directory then, there must be a way to access it from the http port on a browser, so I should check the Apache configuration file
- I founded the Apache server configuration in the `/etc/apache2` directory
- In the config directory, in the `/sites-enabled` directory I found a config file named `internal.conf` exactly as the directory found in `/var/www` 
- When reading it, I found that there is a Virtual Host with the name `internal.openadmin.htb` and the user assigned to it is exactly `joanna` and with this I can execute `/var/www/internal/main.php` to get `joanna` `ssh` key
![[Pasted image 20250830212237.png]]

- I used the next `ssh` command to login as the user jimmy and get access to the Virtual Host `internal` 
```shell
sudo -L 52846:127.0.0.1:52846 jimmy@10.10.10.171
```

- Now when accessing the `127.0.0.1:52846` on the Internet I am met with a login form 
![[Pasted image 20250831204956.png]]
- Now from previously searches on the linux serve we know that we have access to the `/internal` directory inside which are the files 
- The `/var/www/internal` directory contains 3 `php` files: `main.php`, `index.php`, `logout.php` 
- Inside the `index.php` I found the code for the login form and also I found that the user must be `jimmy` and the password must correspond to a certain `sha512` hash
![[Pasted image 20250831205800.png]]
- Using [CrackStation](https://crackstation.net/) I cracked the hash to find the password `Revealed`
![[Pasted image 20250831205929.png]]
- Using the `jimmy` username and the `Revealed` password I successfully logged in and next I got the output of the file `main.php` from the `/internal` directory, in essence I got the output of the command `cat /home/joanna/.ssh/id_rsa`
![[Pasted image 20250831210319.png]]
- I copy the output content and paste it into a `joanna_id_rsa` file on my workstation

- I use the command `ssh -i joanna_id_rsa joanna@10.10.10.171` to try to connect to the `joanna` user account, but I discover that I also need a passphrase for the `joanna_id_rsa` key

- To get the passphrase of the key I am going to use `JohnTheRipper` to crack the hash:
1. First I convert the key into a crackable format
```shell
/usr/share/john/ssh2john.py joanna_id_rsa > joanna_hash.txt
```
2. Crack the newly gen hash using `john` + `rockyou.txt` wordlist file
```shell
john --wordlist=/usr/share/wordlists/rockyou.txt joanna_hash.txt
```
3. Show the cracked password
```shell
john --show joanna_hash.txt
```
![[Pasted image 20250831211711.png]]
- In the end the passphrase I get is `bloodninjas`

- Now I try again to login in to `joanna` user account using the `joanna_id_rsa` key and the `bloodninjas` passphrase and everything worked fine and I got access to `joanna` account and the user flag
![[Pasted image 20250831211904.png]]

## Root Flag

- As usual first command I utilize is `sudo -l` to see If `joanna` can run any command with higher permissions
![[Pasted image 20250831212016.png]]
- It was a success as I found that I can run the `nano` text editor on the `/opt/priv` file

- `nano` comes with a very interesting feature that allows you to execute commands inside the editor, which in our case can let me execute commands as the root user 
- I also found on [GTFObins-nano](https://gtfobins.github.io/gtfobins/nano/) exactly how to get an interactive shell as the `root` user
	1. First open nano with `sudo`: `sudo /bin/nano /opt/priv`
![[Pasted image 20250831213011.png]]
	2. Press CTRL+R followed by CTRL+X `Execute Command`
![[Pasted image 20250831213037.png]]
	3. Type `reset; sh 1>&0 2>&0` and press ENTER
![[Pasted image 20250831213208.png]]
- Everything worked perfectly and I got what I desired, an interactive shell as the `root` user and now I can get the system flag from `/root/root.txt` file
![[Pasted image 20250831213235.png]]
