
- I am being given an IP address, and first thing first I want to enumerate all the open ports on this IP address so I use `nmap` 
```shell
nmap -sC -sV 10.10.10.75
```
![[Pasted image 20250826142005.png]]
- From the result we see that we have 2 open ports:
	- `22` - for `ssh` 
	- `80` - for a web application

- Visiting the web application we are met with an empty page, the only text being `Hello world!` 
- In the meantime I opened `burp suite` and checking the response of the page it points me to the `/nibbleblog` page![[Pasted image 20250826142136.png]]

- Visiting the web page `/nibbleblog` I am shown a home page![[Pasted image 20250826142250.png]]

 - Going through this page doesn't help me very much so the next thing I want to do is to enumerate all the subdirectories and files of the `/nibbleblog` directory 
 - This will be done by using `ffuf` to fuzz for subdirectories
```shell
ffuf -u http://10.10.10.75/nibbleblog/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic -t 150
```
- I get the following output:![[Pasted image 20250826142752.png]]

- From all these subdirectories and files `/nibbleblog/content/private` is very interesting because of some things![[Pasted image 20250826143002.png]]
	1. The server runs `.php` scripts so I can try fuzzing for some hidden `.php` scripts or pages
	2. The file `users.xml`
- The file `users.xml` is interesting because it gives us the username of an existing user, that being the user `admin`![[Pasted image 20250826143154.png]]

- Next thing I use `ffuf` to fuzz for `.php` pages on the `/nibbleblog` directory
```shell
ffuf -u http://10.10.10.75/nibbleblog/FUZZ.php -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -ic -t 150
```
![[Pasted image 20250826143732.png]]

- Very interesting from this search, is the `admin.php` page which contains a login form
![[Pasted image 20250826143912.png]]
- So, let me try to login using the `admin` as username and try different passwords
- Luckily using the password `nibbles` with the `admin` username let me to login and a dashboard page is shown![[Pasted image 20250826144209.png]]


- Going to the settings page, down the page I find the version of `Nibbleblog` which is `4.0.3`
![[Pasted image 20250826144431.png]] 

- Searching on the web for a vulnerability for this version I found the `CVE-2015-6967` which allows Remote Code Execution
- I start the `Metasploit` console to search for an exploit and I find it:
	- `exploit /multi/http/nibbleblog_file_upload`

- I set all the options needed and run the script
	- `RHOSTS`: 10.10.10.75
	- `USERNAME`: admin
	- `PASSWORD`: `nibbles`
	- `TARGETURI`: `/nibbleblog`
![[Pasted image 20250826145444.png]]

- I get a shell as the user `nibbler` and get the `user.txt` flag
![[Pasted image 20250826145640.png]]

## Root flag

- I run the `sudo -l` command to see with what commands I can pair up the `sudo` command
![[Pasted image 20250826145851.png]]
- I can use `sudo` as the `root` user with `/home/nibbler/personal/stuff/monitor.sh` 
- The script does not exist and since the script location should be in the `nibbler` user directory we can create one that gives us a shell as the root user
- We create the script at `/home/nibbler/personal/stuff/monitor.sh`
```shell
echo '#!/bin/bash' > monitor.sh
echo '/bin/bash' >> monitor.sh
```
- Then we give permissions so we can execute it
```shell
chmod 777 monitor.sh
```
- Then we run it
```shell
sudo /home/nibbler/personal/stuff/monitor.sh
```

- After the script was executed, we got a shell as the `root` user and now we can find the `root.txt` flag
![[Pasted image 20250826150943.png]]

