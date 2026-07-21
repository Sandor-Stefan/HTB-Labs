
Given an IP Address `10.10.10.7` I have to test the machine Beep to find the user and system flag of this machine

First I start with some recon and to the best tool to do that is `nmap` to find all the open ports
```shell
nmap -A -T4 10.10.10.7
```
![[Pasted image 20251220114833.png]]

The scan shows a lot of ports open but probably we can't use all of them so 3 of them are standing out, those being:
- port `80` - open for an Apache web server
- port `443` - open for another web server this time on HTTPS but it has an expired SSL certificate
- port `10000` - open for http hosting a `MiniServ 1.570 (Webmin httpd)` server

So now I will use `burpsuite` to check both web servers to see what they are holding

Checking the port 80 of the target leads to nowhere as I get an error when trying to access it

But checking port `443` leads to a login form run by the `Elastix` software
![[Pasted image 20251220122751.png]]

Searching for default credentials on the Internet lead me to nowhere so I decided to try fuzzing the webserver to check for potential exploitable directories
```shell
ffuf -u https://10.10.10.7/FUZZ -w /usr/share/seclists/Discovery/Web-Content/combined_words.txt -ic -t 150
```
![[Pasted image 20251220124730.png]]
The search for hidden directories came in handy as I found a lot of hidden endpoints
- Having these endpoints discovered I checked on the Internet to see if I can find any information about their usage in `Elastix` and if I can exploit in any mean

Searching for the combination of words `Elastix + <hidden_endpoints>` a success was when I searched for the `/vtigercrm` endpoint 
- I found a Local File Inclusion on the `/vtigercrm/graph.php` endpoint which allows me to view files and execute local scripts in the context of the web server process https://www.exploit-db.com/exploits/37637
- The script is very simple and can be executed manually by checking the next URL:
	- `https://10.10.10.7/vtigercrm/graph.php?current_language=../../../../../../../..//etc/amportal.conf%00&module=Accounts&action`
![[Pasted image 20251220130916.png]]
- Through the whole text I found some credentials:
	- username: `asteriskuser`
	- password: `jEhdIekWmdjE`

Next step as I thought I found the credentials of the user, was to ssh into his account but when I tried I was met with this:
![[Pasted image 20251220131102.png]]

Searching on the Internet I fount that I need to input an option when trying to ssh that option being `-oKexAlgorithms=+diffie-hellman-group1-sha1` and now it should work
![[Pasted image 20251220132756.png]]
Of course it was not enough and after another search on the Internet I found that I need to add another option, that being `-oHostKeyAlgorithms=+ssh-rsa`
![[Pasted image 20251220132927.png]]
Now after all the options I entered and the password for that user, it still gave me Permission denied, which is bad but that probably means that the password is for another user

Because there is no wrong in trying the same password but this time for the `root` user I tried that
![[Pasted image 20251220133100.png]]
Guess what it worked for:
	 username - `root`
	 password - `jEhdIekWmdjE`

Now I have access to the entire system which is cool and the only thing left to do is to get the root flag found in the root directory and the user flag found in the `/home/fanis` directory:
![[Pasted image 20251220133341.png]]