
Having the IP address of the box it is time to start cracking it

Inside the `/etc/hosts` file I have to insert: `<IP> devhub.htb`

First I start with port enumeration using `nmap` 
```bash
nmap -Pn -Ss -p- devhub.htb
```
- I use this options to not generate too much noise just to find the open ports
- There are 3 ports available: `22`, `80`, `6274`

Now I enumerate further to get info about the services running on the open ports
```bash
nmap -sV -sC -p22,80,6274 devhub.htb
```
- port `22` is open as usual for SSH, port `80` open for HTTP hosting a website and port `6274` appears to host another website

## Web Enumeration

First we check the website found on port `80`:
![Pasted image 20261008190431](Pasted%20image%2020261008190431.png)
- As we can see the website presents some info about the running services across the server:
	- On port `6274` it actually run Some tool related to MCP servers
	- Also, there is a jupyter-based environment running on the server on port `8888`, but can be accessed only locally
