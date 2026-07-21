
For this machine I was given the `10.10.10.37` IP address

First things first, I start with a `nmap` scan on this IP address for port enumeration
```shell
nmap -A -T4 10.10.10.37
```
![[Pasted image 20251015155336.png]]
The scan reveals 3 open ports:
- port 21 - for FTP version `ProFTPD 1.3.5a`
- port 22 - for SSH
- port 80 - for a http web server

I decide to start checking the web server to see what it contains:
![[Pasted image 20251015155723.png]]
The site name is `BlockyCraft` 

Wandering through the site I found a login page but I also need a username or email and a password
![[Pasted image 20251015160328.png]]

- Now to find any user on the site I check the website one more time and by clicking the `Welcome to BlokcyCraft!` from `Recent Posts` I found the name of a user who posted on the site
![[Pasted image 20251015160539.png]]
- That name is `NOTCH` (surely not a reference to Minecraft :))) )

Now to test if the username `notch` is valid I input it inside the username box in the login form
![[Pasted image 20251015160724.png]]
- After pressing the Log in button I got the confirmation that this is a valid username

- Also there is a `Lost your password` function but when I use it with the `notch` username I find that the server disabled this function
![[Pasted image 20251015161346.png]]

Next step now is to fuzz for hidden directories using `ffuf`
```shell
ffuf -u http://blocky.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 150 -ic
```
The search concluded with a few more directories
![[Pasted image 20251015162354.png]]

From these newly findings the `/plugins` one seems the most interesting as it contains 2 `.jar` files which I decided to download and check them
![[Pasted image 20251015162516.png]]
- After downloading them and unzipping them I get a bunch of files and directories

- I would say the most important one was the `/com` directory which contain a java class file `BlockyCore.class` 
![[Pasted image 20251015165533.png]]

- In the file can be seen 3 lines with interesting content:
	- `localhost`
	- `root`
	- `8YsqfCTnvxAUeduzjNSXe22`
- The last line content `8YsqfCTnvxAUeduzjNSXe22` I believe that is a password and what better place to try it than by using `ssh`

- I try to use `ssh` with the username `notch` and the password found earlier
![[Pasted image 20251015170020.png]]
- This was a success and I was able to access `notch` user account and get the user flag
![[Pasted image 20251015170106.png]]

## Root Flag

Now that I have access as a user I have to try to leverage my privileges to `root` user

- First thing I run is `sudo -l` to see if there is some command I can run as the `root` user
![[Pasted image 20251015170233.png]]
- OH MY OH MY, I can use `sudo` paired with any command (someone has to fire the system admin for this)

- This was pretty easy so all I have to do now is to spawn a shell with `root` access and get the flag
![[Pasted image 20251015170435.png]]