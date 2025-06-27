
- starting with a nmap scan on the IP address
```shell
nmap -sC -sV 10.10.10.245
```

- 3 open ports:
	- 21 - ftp, `vsftpd 3.0.3`
	- 22 - ssh
	- 80 - http, a website called `gunicorn`

- we will try the exploit `CVE-2021-30047` - remote denial of service on `vsftpd 3.0.3`
- no luck in doing that

- we get back to the website
- we enter the website and by browsing it we see that it runs system commands
- we have at page `IP Config` the output of `ifconfig`
- we have at `Network Status` the output of netstat

- at `Security Snapshot` menu item we have a button `Download` which gives us a packet capture file which can be examined using `WireShark` 
- I notice that in the URL when I create a new capture that is of form `/data/<id>` the `<id>` changes for every capture 
- By browsing `/data/0` we reveal a packet capture with a lot of packets
- After downloading the file and going through it we reveal the user credentials ![[Pasted image 20250626205506.png]]
	- username : `nathan` 
	- password: `Buck3tH4TF0RM3!`

- we try them on ssh and we find that they are available and that lets us entering the user account and find the flag

- On the user account we find the script `linepeas.sh` available which helps us with privilege escalation
- we get the output of the script and we search through it
- At `Capabilities` we find that the script `/usr/bin/python3.8` has the settings `cap_setuid` and `cap_net_bind_service`According to the [documentation](https://man7.org/linux/man-pages/man7/capabilities.7.html) `cap_setuid` allows the process to gain `setuid` privileges without the SUID bit set
- This allows to switch to `UID = 0` (root) and we can open a bash with root permissions
- We run python 3.8:
```shell
/usr/bin/python3.8
```
- In the python terminal
```python
import os
os.setuid(0)  / set the UID to 0
os.system("/bin/bash")  / opens a bash
```

- Then we have our root shell and we can go in the root directory to retrieve the flag