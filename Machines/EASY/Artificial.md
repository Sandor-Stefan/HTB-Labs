
- we start with a nmap scan on the address:
```shell
nmap -sC -sV 10.10.11.74
```

- We have 3 open ports:
	- port `22` ssh:
		- OpenSSH `8.2p1`
	- port `80` http:
		- running on `nginx 1.18.0`
		- `http://artificial.htb/`
	- port `8000` python http server:
		- we have a SimpleHTTPServer 0.6 on python 3.8.10

- We try the web server:
- we write in `/etc/hosts` 
```txt
10.10.11.74 artificial.htb
```

- we open the site and we have 4 tabs
- on 2 of them `Login` and `Register` we have login forms 
- by registering and after login in the same account we are met with a new page `Dashboard` where we can upload files

