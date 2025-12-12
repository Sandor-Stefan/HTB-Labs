
As usually the goal is to find a user and a system flag from a target, having only the target IP address

**Recon**

I am starting with an nmap scan to uncover the open ports and their service
![[Pasted image 20251206181127.png]]
The scan shows 2 open ports:
 - port 22 - open for ssh
 - port 80 - open for http, hosting an Apache web server

Checking the web server I am met with the Apache2 Default Page meaning that if there is something on the webserver it must be hidden
![[Pasted image 20251206181543.png]]

To check if there are any hidden endpoints, fuzzing the website would be the solution
- To fuzz the website I will use `ffuf` to do that
```shell
ffuf -u http://10.10.11.48/FUZZ -w /usr/share/seclists/Discovery/Web-Content/combined_directories.txt -ic -t 150
```
![[Pasted image 20251206183553.png]]
The search was positive as I found a hidden directory `/daloradius` but I can't access it directly and I need to do another fuzz on this directory

Fuzzing the directory `/daloradius` I found some entries from which the `/app` entry was for me an interesting new target
![[Pasted image 20251206184154.png]]

After fuzzing on the new directory `/daloradius/app` I found 3 new endpoints which from the name I think that they might hold something
![[Pasted image 20251206184716.png]]

Checking the `/daloradius/app/users` I am met with a login page
![[Pasted image 20251206185140.png]]

Checking the `/daloradius/app/operators` I am met with another login page, but this time I also can see the version of `daloradius 2.2 beta`
![[Pasted image 20251206185256.png]]

Searching on Google for something related to this version I didn't find any vulnerability that could be useful but I found some default credentials:
	- username: `administrator`
	- password: `radius`
Entering the credentials in, I successfully logged in and now I can see the home page
![[Pasted image 20251206185723.png]]

Clicking on the `Go to users list` I am redirected to another page where I can see the only user present `svcMosh` and its password hashed
![[Pasted image 20251206190243.png]]

Next I went to https://crackstation.net/ to crack the hash and get the password
![[Pasted image 20251206190320.png]]

Now that I have a user and a password I can login to ssh with the following credentials:
- username: `svcMosh`
- password: `underwaterfriends`
The credentials worked and now I gained access to the server as the `svcMosh` user and also found the user flag
![[Pasted image 20251206190606.png]]

# System Flag

To start the escalation process I ran the `sudo -l` command to see what I can run with `sudo` privileges:
![[Pasted image 20251206190958.png]]
The result was that I can run the `sudo /usr/bin/mosh-server` command 

`Mosh-server` works like ssh, but it is used for mobiles, so when we run `sudo /usr/bin/mosh-server` we start a mosh server with `root` permissions to which we can connect and gain unwanted access

1. We run `sudo /usr/bin/mosh-server`
![[Pasted image 20251206193158.png]]
- We can see from the output that server started and gave us some information about connection:
	- `60001` - represents the port on which we have to make the connection on localhost
	- `Zt28CeTGwPQNO721QFvEWg` - represents the `MOSH_KEY` which we have to export in order to gain access
1. Next step is to run `export MOSH_KEY='Zt28CeTGwPQNO721QFvEWg'`
2. Finally we can use `mosh-client 127.0.0.1 60001` to connect to the server
![[Pasted image 20251206193429.png]]

Everything went as it should and I gained access as the `root` user and now I can access the system flag from `root.txt`
![[Pasted image 20251206193112.png]]